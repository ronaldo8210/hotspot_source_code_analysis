在YGC阶段，需要遍历每个年轻代的Remembered Set，根据Remembered Set的记录找到引用了年轻代对象的老年代对象，再以找到的老年代对象为根去深度遍历存活的年轻代对象。

HeapRegionRemSetIterator类负责遍历Remembered Set的工作，主要成员变量有：

```c++
class HeapRegionRemSetIterator : public StackObj {
 private:
  // The region RSet over which we are iterating.
  HeapRegionRemSet* _hrrs;

  // Local caching of HRRS fields.
  const BitMap*             _coarse_map;

  G1BlockOffsetSharedArray* _bosa;
  G1CollectedHeap*          _g1h;

  // The number of cards yielded since initialization.
  size_t _n_yielded_fine;
  size_t _n_yielded_coarse;
  size_t _n_yielded_sparse;

  // Indicates what granularity of table that we are currently iterating over.
  // We start iterating over the sparse table, progress to the fine grain
  // table, and then finish with the coarse table.
  enum IterState {
    Sparse,
    Fine,
    Coarse
  };
  IterState _is;

  // For both Coarse and Fine remembered set iteration this contains the
  // first card number of the heap region we currently iterate over.
  size_t _cur_region_card_offset;

  // Current region index for the Coarse remembered set iteration.
  int    _coarse_cur_region_index;
  size_t _coarse_cur_region_cur_card;

  // The PRT we are currently iterating over.
  PerRegionTable* _fine_cur_prt;
  // Card offset within the current PRT.
  size_t _cur_card_in_prt;

  // The Sparse remembered set iterator.
  SparsePRTIter _sparse_iter;

};
```

遍历一个Remembered Set的入口函数为HeapRegionRemSetIterator::has_next()：
```c++
bool HeapRegionRemSetIterator::has_next(size_t& card_index) {
  switch (_is) {
  case Sparse: {
    if (_sparse_iter.has_next(card_index)) {
      _n_yielded_sparse++;
      return true;
    }
    // Otherwise, deliberate fall-through
    _is = Fine;
    PerRegionTable* initial_fine_prt = _hrrs->_other_regions._first_all_fine_prts;
    if (initial_fine_prt != NULL) {
      switch_to_prt(_hrrs->_other_regions._first_all_fine_prts);
    }
  }
  case Fine:
    if (fine_has_next(card_index)) {
      _n_yielded_fine++;
      return true;
    }
    // Otherwise, deliberate fall-through
    _is = Coarse;
  case Coarse:
    if (coarse_has_next(card_index)) {
      _n_yielded_coarse++;
      return true;
    }
    // Otherwise...
    break;
  }
  assert(ParallelGCThreads > 1 ||
         n_yielded() == _hrrs->occupied(),
         "Should have yielded all the cards in the rem set "
         "(in the non-par case).");
  return false;
}
```

RSHashTableIter::has_next()负责遍历sparsePRT结构的引用记录：
```c++
bool RSHashTableIter::has_next(size_t& card_index) {
  _card_ind++;
  CardIdx_t ci;
  if (_card_ind < SparsePRTEntry::cards_num() &&
      ((ci = _rsht->entry(_bl_ind)->card(_card_ind)) !=
       SparsePRTEntry::NullEntry)) {
    card_index = compute_card_ind(ci);
    return true;
  }
  // Otherwise, must find the next valid entry.
  _card_ind = 0;

  if (_bl_ind != RSHashTable::NullEntry) {
      _bl_ind = _rsht->entry(_bl_ind)->next_index();
      ci = find_first_card_in_list();
      if (ci != SparsePRTEntry::NullEntry) {
        card_index = compute_card_ind(ci);
        return true;
      }
  }
  // If we didn't return above, must go to the next non-null table index.
  _tbl_ind++;
  while ((size_t)_tbl_ind < _rsht->capacity()) {
    _bl_ind = _rsht->_buckets[_tbl_ind];
    ci = find_first_card_in_list();
    if (ci != SparsePRTEntry::NullEntry) {
      card_index = compute_card_ind(ci);
      return true;
    }
    // Otherwise, try next entry.
    _tbl_ind++;
  }
  // Otherwise, there were no entry.
  return false;
}
```

HeapRegionRemSetIterator::fine_has_next()负责遍历PerRegionTable结构的引用记录：
```c++
bool HeapRegionRemSetIterator::fine_has_next(size_t& card_index) {
  if (fine_has_next()) {
    _cur_card_in_prt =
      _fine_cur_prt->_bm.get_next_one_offset(_cur_card_in_prt + 1);
  }
  if (_cur_card_in_prt == HeapRegion::CardsPerRegion) {
    // _fine_cur_prt may still be NULL in case if there are not PRTs at all for
    // the remembered set.
    if (_fine_cur_prt == NULL || _fine_cur_prt->next() == NULL) {
      return false;
    }
    PerRegionTable* next_prt = _fine_cur_prt->next();
    switch_to_prt(next_prt);
    _cur_card_in_prt = _fine_cur_prt->_bm.get_next_one_offset(_cur_card_in_prt + 1);
  }

  card_index = _cur_region_card_offset + _cur_card_in_prt;
  guarantee(_cur_card_in_prt < HeapRegion::CardsPerRegion,
            err_msg("Card index "SIZE_FORMAT" must be within the region", _cur_card_in_prt));
  return true;
}
```

HeapRegionRemSetIterator::coarse_has_next()负责遍历CoarseMap结构的引用记录：
```c++
bool HeapRegionRemSetIterator::coarse_has_next(size_t& card_index) {
  if (_hrrs->_other_regions._n_coarse_entries == 0) return false;
  // Go to the next card.
  _coarse_cur_region_cur_card++;
  // Was the last the last card in the current region?
  if (_coarse_cur_region_cur_card == HeapRegion::CardsPerRegion) {
    // Yes: find the next region.  This may leave _coarse_cur_region_index
    // Set to the last index, in which case there are no more coarse
    // regions.
    _coarse_cur_region_index =
      (int) _coarse_map->get_next_one_offset(_coarse_cur_region_index + 1);
    if ((size_t)_coarse_cur_region_index < _coarse_map->size()) {
      _coarse_cur_region_cur_card = 0;
      HeapWord* r_bot =
        _g1h->region_at((uint) _coarse_cur_region_index)->bottom();
      _cur_region_card_offset = _bosa->index_for(r_bot);
    } else {
      return false;
    }
  }
  // If we didn't return false above, then we can yield a card.
  card_index = _cur_region_card_offset + _coarse_cur_region_cur_card;
  return true;
}
```
