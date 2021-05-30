[RSet异步更新原理](#jump_1)

[Refine线程源码解析](#jump_2)

[Refine线程工作过程内存变迁](#jump_3)

## <span id="jump_1">RSet异步更新原理</span>


## <span id="jump_2">Refine线程源码解析</span>
虚拟机启动后，N个Refine线程随之启动，工作入口函数在ConcurrentG1RefineThread::run()，工作过程函数调用链为：

  - ConcurrentG1RefineThread::run()
  - DirtyCardQueueSet::apply_closure_to_completed_buffer()
  - DirtyCardQueueSet::apply_closure_to_completed_buffer_helper()
  - DirtyCardQueue::apply_closure_to_buffer()
  - RefineCardTableEntryClosure::do_card_ptr()
  - G1RemSet::refine_card()
  - HeapRegion::oops_on_card_seq_iterate_careful()
  - 

Refine线程工作过程主要函数源码解析如下：

  - ConcurrentG1RefineThread::run()
  ```c++
  void ConcurrentG1RefineThread::run() {
    initialize_in_thread();
    wait_for_universe_init();

    if (_worker_id >= cg1r()->worker_thread_num()) {
      run_young_rs_sampling();
      terminate();
      return;
    }

    _vtime_start = os::elapsedVTime();
    while (!_should_terminate) {
      DirtyCardQueueSet& dcqs = JavaThread::dirty_card_queue_set();

      // Wait for work
      wait_for_completed_buffers();

      if (_should_terminate) {
        break;
      }

      {
        SuspendibleThreadSetJoiner sts;

        do {
          int curr_buffer_num = (int)dcqs.completed_buffers_num();
          // If the number of the buffers falls down into the yellow zone,
          // that means that the transition period after the evacuation pause has ended.
          if (dcqs.completed_queue_padding() > 0 && curr_buffer_num <= cg1r()->yellow_zone()) {
            dcqs.set_completed_queue_padding(0);
          }

          if (_worker_id > 0 && curr_buffer_num <= _deactivation_threshold) {
            // If the number of the buffer has fallen below our threshold
            // we should deactivate. The predecessor will reactivate this
            // thread should the number of the buffers cross the threshold again.
            deactivate();
            break;
          }

          // Check if we need to activate the next thread.
          if (_next != NULL && !_next->is_active() && curr_buffer_num > _next->_threshold) {
            _next->activate();
          }
        } while (dcqs.apply_closure_to_completed_buffer(_refine_closure, _worker_id + _worker_id_offset, cg1r()->green_zone()));

        // We can exit the loop above while being active if there was a yield request.
        if (is_active()) {
          deactivate();
        }
      }

      if (os::supports_vtime()) {
        _vtime_accum = (os::elapsedVTime() - _vtime_start);
      } else {
        _vtime_accum = 0.0;
      }
    }
    assert(_should_terminate, "just checking");
    terminate();
  }
  ```
  
  - DirtyCardQueueSet::apply_closure_to_completed_buffer()
  ```c++
  bool DirtyCardQueueSet::apply_closure_to_completed_buffer(CardTableEntryClosure* cl,
                                                            uint worker_i,
                                                            int stop_at,
                                                            bool during_pause) {
    assert(!during_pause || stop_at == 0, "Should not leave any completed buffers during a pause");
    BufferNode* nd = get_completed_buffer(stop_at);
    bool res = apply_closure_to_completed_buffer_helper(cl, worker_i, nd);
    if (res) Atomic::inc(&_processed_buffers_rs_thread);
    return res;
  }
  ```
  
  - DirtyCardQueueSet::apply_closure_to_completed_buffer_helper()
  ```c++
  bool DirtyCardQueueSet::
  apply_closure_to_completed_buffer_helper(CardTableEntryClosure* cl,
                                           uint worker_i,
                                           BufferNode* nd) {
    if (nd != NULL) {
      void **buf = BufferNode::make_buffer_from_node(nd);
      size_t index = nd->index();
      bool b =
        DirtyCardQueue::apply_closure_to_buffer(cl, buf,
                                                index, _sz,
                                                true, worker_i);
      if (b) {
        deallocate_buffer(buf);
        return true;  // In normal case, go on to next buffer.
      } else {
        enqueue_complete_buffer(buf, index);
        return false;
      }
    } else {
      return false;
    }
  }
  ```
  
  - DirtyCardQueue::apply_closure_to_buffer()
  ```c++
  bool DirtyCardQueue::apply_closure_to_buffer(CardTableEntryClosure* cl,
                                               void** buf,
                                               size_t index, size_t sz,
                                               bool consume,
                                               uint worker_i) {
    if (cl == NULL) return true;
    for (size_t i = index; i < sz; i += oopSize) {
      int ind = byte_index_to_index((int)i);
      jbyte* card_ptr = (jbyte*)buf[ind];
      if (card_ptr != NULL) {
        // Set the entry to null, so we don't do it again (via the test
        // above) if we reconsider this buffer.
        if (consume) buf[ind] = NULL;
        if (!cl->do_card_ptr(card_ptr, worker_i)) return false;
      }
    }
    return true;
  }
  ```
  
  - RefineCardTableEntryClosure::do_card_ptr()
  ```c++
  bool do_card_ptr(jbyte* card_ptr, uint worker_i) {
    bool oops_into_cset = G1CollectedHeap::heap()->g1_rem_set()->refine_card(card_ptr, worker_i, false);
    // This path is executed by the concurrent refine or mutator threads,
    // concurrently, and so we do not care if card_ptr contains references
    // that point into the collection set.
    assert(!oops_into_cset, "should be");

    if (_concurrent && SuspendibleThreadSet::should_yield()) {
      // Caller will actually yield.
      return false;
    }
    // Otherwise, we finished successfully; return true.
    return true;
  }
  ```
  
  - G1RemSet::refine_card()
  ```c++
  bool G1RemSet::refine_card(jbyte* card_ptr, uint worker_i,
                             bool check_for_refs_into_cset) {
    assert(_g1->is_in_exact(_ct_bs->addr_for(card_ptr)),
           err_msg("Card at "PTR_FORMAT" index "SIZE_FORMAT" representing heap at "PTR_FORMAT" (%u) must be in committed heap",
                   p2i(card_ptr),
                   _ct_bs->index_for(_ct_bs->addr_for(card_ptr)),
                   _ct_bs->addr_for(card_ptr),
                   _g1->addr_to_region(_ct_bs->addr_for(card_ptr))));

    // If the card is no longer dirty, nothing to do.
    if (*card_ptr != CardTableModRefBS::dirty_card_val()) {
      // No need to return that this card contains refs that point
      // into the collection set.
      return false;
    }

    // Construct the region representing the card.
    HeapWord* start = _ct_bs->addr_for(card_ptr);
    // And find the region containing it.
    HeapRegion* r = _g1->heap_region_containing(start);

    // Why do we have to check here whether a card is on a young region,
    // given that we dirty young regions and, as a result, the
    // post-barrier is supposed to filter them out and never to enqueue
    // them? When we allocate a new region as the "allocation region" we
    // actually dirty its cards after we release the lock, since card
    // dirtying while holding the lock was a performance bottleneck. So,
    // as a result, it is possible for other threads to actually
    // allocate objects in the region (after the acquire the lock)
    // before all the cards on the region are dirtied. This is unlikely,
    // and it doesn't happen often, but it can happen. So, the extra
    // check below filters out those cards.
    if (r->is_young()) {
      return false;
    }

    // While we are processing RSet buffers during the collection, we
    // actually don't want to scan any cards on the collection set,
    // since we don't want to update remebered sets with entries that
    // point into the collection set, given that live objects from the
    // collection set are about to move and such entries will be stale
    // very soon. This change also deals with a reliability issue which
    // involves scanning a card in the collection set and coming across
    // an array that was being chunked and looking malformed. Note,
    // however, that if evacuation fails, we have to scan any objects
    // that were not moved and create any missing entries.
    if (r->in_collection_set()) {
      return false;
    }

    // The result from the hot card cache insert call is either:
    //   * pointer to the current card
    //     (implying that the current card is not 'hot'),
    //   * null
    //     (meaning we had inserted the card ptr into the "hot" card cache,
    //     which had some headroom),
    //   * a pointer to a "hot" card that was evicted from the "hot" cache.
    //

    G1HotCardCache* hot_card_cache = _cg1r->hot_card_cache();
    if (hot_card_cache->use_cache()) {
      assert(!check_for_refs_into_cset, "sanity");
      assert(!SafepointSynchronize::is_at_safepoint(), "sanity");

      card_ptr = hot_card_cache->insert(card_ptr);
      if (card_ptr == NULL) {
        // There was no eviction. Nothing to do.
        return false;
      }

      start = _ct_bs->addr_for(card_ptr);
      r = _g1->heap_region_containing(start);

      // Checking whether the region we got back from the cache
      // is young here is inappropriate. The region could have been
      // freed, reallocated and tagged as young while in the cache.
      // Hence we could see its young type change at any time.
    }

    // Don't use addr_for(card_ptr + 1) which can ask for
    // a card beyond the heap.  This is not safe without a perm
    // gen at the upper end of the heap.
    HeapWord* end   = start + CardTableModRefBS::card_size_in_words;
    MemRegion dirtyRegion(start, end);

  #if CARD_REPEAT_HISTO
    init_ct_freq_table(_g1->max_capacity());
    ct_freq_note_card(_ct_bs->index_for(start));
  #endif

    G1ParPushHeapRSClosure* oops_in_heap_closure = NULL;
    if (check_for_refs_into_cset) {
      // ConcurrentG1RefineThreads have worker numbers larger than what
      // _cset_rs_update_cl[] is set up to handle. But those threads should
      // only be active outside of a collection which means that when they
      // reach here they should have check_for_refs_into_cset == false.
      assert((size_t)worker_i < n_workers(), "index of worker larger than _cset_rs_update_cl[].length");
      oops_in_heap_closure = _cset_rs_update_cl[worker_i];
    }
    G1UpdateRSOrPushRefOopClosure update_rs_oop_cl(_g1,
                                                   _g1->g1_rem_set(),
                                                   oops_in_heap_closure,
                                                   check_for_refs_into_cset,
                                                   worker_i);
    update_rs_oop_cl.set_from(r);

    G1TriggerClosure trigger_cl;
    FilterIntoCSClosure into_cs_cl(NULL, _g1, &trigger_cl);
    G1InvokeIfNotTriggeredClosure invoke_cl(&trigger_cl, &into_cs_cl);
    G1Mux2Closure mux(&invoke_cl, &update_rs_oop_cl);

    FilterOutOfRegionClosure filter_then_update_rs_oop_cl(r,
                          (check_for_refs_into_cset ?
                                  (OopClosure*)&mux :
                                  (OopClosure*)&update_rs_oop_cl));

    // The region for the current card may be a young region. The
    // current card may have been a card that was evicted from the
    // card cache. When the card was inserted into the cache, we had
    // determined that its region was non-young. While in the cache,
    // the region may have been freed during a cleanup pause, reallocated
    // and tagged as young.
    //
    // We wish to filter out cards for such a region but the current
    // thread, if we're running concurrently, may "see" the young type
    // change at any time (so an earlier "is_young" check may pass or
    // fail arbitrarily). We tell the iteration code to perform this
    // filtering when it has been determined that there has been an actual
    // allocation in this region and making it safe to check the young type.
    bool filter_young = true;

    HeapWord* stop_point =
      r->oops_on_card_seq_iterate_careful(dirtyRegion,
                                          &filter_then_update_rs_oop_cl,
                                          filter_young,
                                          card_ptr);

    // If stop_point is non-null, then we encountered an unallocated region
    // (perhaps the unfilled portion of a TLAB.)  For now, we'll dirty the
    // card and re-enqueue: if we put off the card until a GC pause, then the
    // unallocated portion will be filled in.  Alternatively, we might try
    // the full complexity of the technique used in "regular" precleaning.
    if (stop_point != NULL) {
      // The card might have gotten re-dirtied and re-enqueued while we
      // worked.  (In fact, it's pretty likely.)
      if (*card_ptr != CardTableModRefBS::dirty_card_val()) {
        *card_ptr = CardTableModRefBS::dirty_card_val();
        MutexLockerEx x(Shared_DirtyCardQ_lock,
                        Mutex::_no_safepoint_check_flag);
        DirtyCardQueue* sdcq =
          JavaThread::dirty_card_queue_set().shared_dirty_card_queue();
        sdcq->enqueue(card_ptr);
      }
    } else {
      _conc_refine_cards++;
    }

    // This gets set to true if the card being refined has
    // references that point into the collection set.
    bool has_refs_into_cset = trigger_cl.triggered();

    // We should only be detecting that the card contains references
    // that point into the collection set if the current thread is
    // a GC worker thread.
    assert(!has_refs_into_cset || SafepointSynchronize::is_at_safepoint(),
             "invalid result at non safepoint");

    return has_refs_into_cset;
  }
  ```
  
  - HeapRegion::oops_on_card_seq_iterate_careful()
  ```c++
  HeapWord*
  HeapRegion::
  oops_on_card_seq_iterate_careful(MemRegion mr,
                                   FilterOutOfRegionClosure* cl,
                                   bool filter_young,
                                   jbyte* card_ptr) {
    // Currently, we should only have to clean the card if filter_young
    // is true and vice versa.
    if (filter_young) {
      assert(card_ptr != NULL, "pre-condition");
    } else {
      assert(card_ptr == NULL, "pre-condition");
    }
    G1CollectedHeap* g1h = G1CollectedHeap::heap();

    // If we're within a stop-world GC, then we might look at a card in a
    // GC alloc region that extends onto a GC LAB, which may not be
    // parseable.  Stop such at the "scan_top" of the region.
    if (g1h->is_gc_active()) {
      mr = mr.intersection(MemRegion(bottom(), scan_top()));
    } else {
      mr = mr.intersection(used_region());
    }
    if (mr.is_empty()) return NULL;
    // Otherwise, find the obj that extends onto mr.start().

    // The intersection of the incoming mr (for the card) and the
    // allocated part of the region is non-empty. This implies that
    // we have actually allocated into this region. The code in
    // G1CollectedHeap.cpp that allocates a new region sets the
    // is_young tag on the region before allocating. Thus we
    // safely know if this region is young.
    if (is_young() && filter_young) {
      return NULL;
    }

    assert(!is_young(), "check value of filter_young");

    // We can only clean the card here, after we make the decision that
    // the card is not young. And we only clean the card if we have been
    // asked to (i.e., card_ptr != NULL).
    if (card_ptr != NULL) {
      *card_ptr = CardTableModRefBS::clean_card_val();
      // We must complete this write before we do any of the reads below.
      OrderAccess::storeload();
    }

    // Cache the boundaries of the memory region in some const locals
    HeapWord* const start = mr.start();
    HeapWord* const end = mr.end();

    // We used to use "block_start_careful" here.  But we're actually happy
    // to update the BOT while we do this...
    HeapWord* cur = block_start(start);
    assert(cur <= start, "Postcondition");

    oop obj;

    HeapWord* next = cur;
    do {
      cur = next;
      obj = oop(cur);
      if (obj->klass_or_null() == NULL) {
        // Ran into an unparseable point.
        return cur;
      }
      // Otherwise...
      next = cur + block_size(cur);
    } while (next <= start);

    // If we finish the above loop...We have a parseable object that
    // begins on or before the start of the memory region, and ends
    // inside or spans the entire region.
    assert(cur <= start, "Loop postcondition");
    assert(obj->klass_or_null() != NULL, "Loop postcondition");

    do {
      obj = oop(cur);
      assert((cur + block_size(cur)) > (HeapWord*)obj, "Loop invariant");
      if (obj->klass_or_null() == NULL) {
        // Ran into an unparseable point.
        return cur;
      }

      // Advance the current pointer. "obj" still points to the object to iterate.
      cur = cur + block_size(cur);

      if (!g1h->is_obj_dead(obj)) {
        // Non-objArrays are sometimes marked imprecise at the object start. We
        // always need to iterate over them in full.
        // We only iterate over object arrays in full if they are completely contained
        // in the memory region.
        if (!obj->is_objArray() || (((HeapWord*)obj) >= start && cur <= end)) {
          obj->oop_iterate(cl);
        } else {
          obj->oop_iterate(cl, mr);
        }
      }
    } while (cur < end);

    return NULL;
  }
  ```
  
  - 
## <span id="jump_3">Refine线程工作过程内存变迁</span>
