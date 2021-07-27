在Java程序运行过程中，老年代对象引用一个年轻代对象时，要在年轻代对象所处的Region的Remembered Set中记录该引用关系，入口在OtherRegionsTable::add_reference()：
```c++
void OtherRegionsTable::add_reference(OopOrNarrowOopStar from, int tid) {
  uint cur_hrm_ind = hr()->hrm_index();

  if (G1TraceHeapRegionRememberedSet) {
    gclog_or_tty->print_cr("ORT::add_reference_work(" PTR_FORMAT "->" PTR_FORMAT ").",
                                                    from,
                                                    UseCompressedOops
                                                    ? (void *)oopDesc::load_decode_heap_oop((narrowOop*)from)
                                                    : (void *)oopDesc::load_decode_heap_oop((oop*)from));
  }

  int from_card = (int)(uintptr_t(from) >> CardTableModRefBS::card_shift);

  if (G1TraceHeapRegionRememberedSet) {
    gclog_or_tty->print_cr("Table for [" PTR_FORMAT "...): card %d (cache = "INT32_FORMAT")",
                  hr()->bottom(), from_card,
                  FromCardCache::at((uint)tid, cur_hrm_ind));
  }

  if (FromCardCache::contains_or_replace((uint)tid, cur_hrm_ind, from_card)) {
    if (G1TraceHeapRegionRememberedSet) {
      gclog_or_tty->print_cr("  from-card cache hit.");
    }
    assert(contains_reference(from), "We just added it!");
    return;
  }

  // Note that this may be a continued H region.
  HeapRegion* from_hr = _g1h->heap_region_containing_raw(from);
  RegionIdx_t from_hrm_ind = (RegionIdx_t) from_hr->hrm_index();

  // If the region is already coarsened, return.
  if (_coarse_map.at(from_hrm_ind)) {
    if (G1TraceHeapRegionRememberedSet) {
      gclog_or_tty->print_cr("  coarse map hit.");
    }
    assert(contains_reference(from), "We just added it!");
    return;
  }

  // Otherwise find a per-region table to add it to.
  size_t ind = from_hrm_ind & _mod_max_fine_entries_mask;
  PerRegionTable* prt = find_region_table(ind, from_hr);
  if (prt == NULL) {
    MutexLockerEx x(_m, Mutex::_no_safepoint_check_flag);
    // Confirm that it's really not there...
    prt = find_region_table(ind, from_hr);
    if (prt == NULL) {

      uintptr_t from_hr_bot_card_index =
        uintptr_t(from_hr->bottom())
          >> CardTableModRefBS::card_shift;
      CardIdx_t card_index = from_card - from_hr_bot_card_index;
      assert(0 <= card_index && (size_t)card_index < HeapRegion::CardsPerRegion,
             "Must be in range.");
      if (G1HRRSUseSparseTable &&
          _sparse_table.add_card(from_hrm_ind, card_index)) {
        if (G1RecordHRRSOops) {
          HeapRegionRemSet::record(hr(), from);
          if (G1TraceHeapRegionRememberedSet) {
            gclog_or_tty->print("   Added card " PTR_FORMAT " to region "
                                "[" PTR_FORMAT "...) for ref " PTR_FORMAT ".\n",
                                align_size_down(uintptr_t(from),
                                                CardTableModRefBS::card_size),
                                hr()->bottom(), from);
          }
        }
        if (G1TraceHeapRegionRememberedSet) {
          gclog_or_tty->print_cr("   added card to sparse table.");
        }
        assert(contains_reference_locked(from), "We just added it!");
        return;
      } else {
        if (G1TraceHeapRegionRememberedSet) {
          gclog_or_tty->print_cr("   [tid %d] sparse table entry "
                        "overflow(f: %d, t: %u)",
                        tid, from_hrm_ind, cur_hrm_ind);
        }
      }

      if (_n_fine_entries == _max_fine_entries) {
        prt = delete_region_table();
        // There is no need to clear the links to the 'all' list here:
        // prt will be reused immediately, i.e. remain in the 'all' list.
        prt->init(from_hr, false /* clear_links_to_all_list */);
      } else {
        prt = PerRegionTable::alloc(from_hr);
        link_to_all(prt);
      }

      PerRegionTable* first_prt = _fine_grain_regions[ind];
      prt->set_collision_list_next(first_prt);
      _fine_grain_regions[ind] = prt;
      _n_fine_entries++;

      if (G1HRRSUseSparseTable) {
        // Transfer from sparse to fine-grain.
        SparsePRTEntry *sprt_entry = _sparse_table.get_entry(from_hrm_ind);
        assert(sprt_entry != NULL, "There should have been an entry");
        for (int i = 0; i < SparsePRTEntry::cards_num(); i++) {
          CardIdx_t c = sprt_entry->card(i);
          if (c != SparsePRTEntry::NullEntry) {
            prt->add_card(c);
          }
        }
        // Now we can delete the sparse entry.
        bool res = _sparse_table.delete_entry(from_hrm_ind);
        assert(res, "It should have been there.");
      }
    }
    assert(prt != NULL && prt->hr() == from_hr, "consequence");
  }
  // Note that we can't assert "prt->hr() == from_hr", because of the
  // possibility of concurrent reuse.  But see head comment of
  // OtherRegionsTable for why this is OK.
  assert(prt != NULL, "Inv");

  prt->add_reference(from);

  if (G1RecordHRRSOops) {
    HeapRegionRemSet::record(hr(), from);
    if (G1TraceHeapRegionRememberedSet) {
      gclog_or_tty->print("Added card " PTR_FORMAT " to region "
                          "[" PTR_FORMAT "...) for ref " PTR_FORMAT ".\n",
                          align_size_down(uintptr_t(from),
                                          CardTableModRefBS::card_size),
                          hr()->bottom(), from);
    }
  }
  assert(contains_reference(from), "We just added it!");
}
```
