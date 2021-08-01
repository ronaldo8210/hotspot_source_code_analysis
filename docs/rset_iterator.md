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

  bool coarse_has_next(size_t& card_index);

  // The PRT we are currently iterating over.
  PerRegionTable* _fine_cur_prt;
  // Card offset within the current PRT.
  size_t _cur_card_in_prt;

  // Update internal variables when switching to the given PRT.
  void switch_to_prt(PerRegionTable* prt);
  bool fine_has_next();
  bool fine_has_next(size_t& card_index);

  // The Sparse remembered set iterator.
  SparsePRTIter _sparse_iter;

 public:
  HeapRegionRemSetIterator(HeapRegionRemSet* hrrs);

  // If there remains one or more cards to be yielded, returns true and
  // sets "card_index" to one of those cards (which is then considered
  // yielded.)   Otherwise, returns false (and leaves "card_index"
  // undefined.)
  bool has_next(size_t& card_index);

  size_t n_yielded_fine() { return _n_yielded_fine; }
  size_t n_yielded_coarse() { return _n_yielded_coarse; }
  size_t n_yielded_sparse() { return _n_yielded_sparse; }
  size_t n_yielded() {
    return n_yielded_fine() + n_yielded_coarse() + n_yielded_sparse();
  }
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

