[线程私有空间分配对象原理](#jump_1)

[源码解析](#jump_2)

## <span id="jump_1">线程私有空间分配对象原理</span>



## <span id="jump_2">源码解析</span>

1. 数据结构

  主要的数据结构为ThreadLocalAllocBuffer对象：
  
  ```c++
  class ThreadLocalAllocBuffer: public CHeapObj<mtThread> {
    friend class VMStructs;
  private:
    HeapWord* _start;                              // address of TLAB
    HeapWord* _top;                                // address after last allocation
    HeapWord* _pf_top;                             // allocation prefetch watermark
    HeapWord* _end;                                // allocation end (excluding alignment_reserve)
    size_t    _desired_size;                       // desired size   (including alignment_reserve)
    size_t    _refill_waste_limit;                 // hold onto tlab if free() is larger than this
    size_t    _allocated_before_last_gc;           // total bytes allocated up until the last gc

    static size_t   _max_size;                     // maximum size of any TLAB
    static unsigned _target_refills;               // expected number of refills between GCs

    unsigned  _number_of_refills;
    unsigned  _fast_refill_waste;
    unsigned  _slow_refill_waste;
    unsigned  _gc_waste;
    unsigned  _slow_allocations;

    AdaptiveWeightedAverage _allocation_fraction;  // fraction of eden allocated in tlabs

    void accumulate_statistics();
    void initialize_statistics();

    void set_start(HeapWord* start)                { _start = start; }
    void set_end(HeapWord* end)                    { _end = end; }
    void set_top(HeapWord* top)                    { _top = top; }
    void set_pf_top(HeapWord* pf_top)              { _pf_top = pf_top; }
    void set_desired_size(size_t desired_size)     { _desired_size = desired_size; }
    void set_refill_waste_limit(size_t waste)      { _refill_waste_limit = waste;  }

    size_t initial_refill_waste_limit()            { return desired_size() / TLABRefillWasteFraction; }

    static int    target_refills()                 { return _target_refills; }
    size_t initial_desired_size();

    size_t remaining() const                       { return end() == NULL ? 0 : pointer_delta(hard_end(), top()); }

    // Make parsable and release it.
    void reset();

    // Resize based on amount of allocation, etc.
    void resize();

    void invariants() const { assert(top() >= start() && top() <= end(), "invalid tlab"); }

    void initialize(HeapWord* start, HeapWord* top, HeapWord* end);

    void print_stats(const char* tag);

    Thread* myThread();

    // statistics

    int number_of_refills() const { return _number_of_refills; }
    int fast_refill_waste() const { return _fast_refill_waste; }
    int slow_refill_waste() const { return _slow_refill_waste; }
    int gc_waste() const          { return _gc_waste; }
    int slow_allocations() const  { return _slow_allocations; }

    static GlobalTLABStats* _global_stats;
    static GlobalTLABStats* global_stats() { return _global_stats; }

  public:
    ThreadLocalAllocBuffer() : _allocation_fraction(TLABAllocationWeight), _allocated_before_last_gc(0) {
      // do nothing.  tlabs must be inited by initialize() calls
    }

    static const size_t min_size()                 { return align_object_size(MinTLABSize / HeapWordSize) + alignment_reserve(); }
    static const size_t max_size()                 { assert(_max_size != 0, "max_size not set up"); return _max_size; }
    static void set_max_size(size_t max_size)      { _max_size = max_size; }

    HeapWord* start() const                        { return _start; }
    HeapWord* end() const                          { return _end; }
    HeapWord* hard_end() const                     { return _end + alignment_reserve(); }
    HeapWord* top() const                          { return _top; }
    HeapWord* pf_top() const                       { return _pf_top; }
    size_t desired_size() const                    { return _desired_size; }
    size_t used() const                            { return pointer_delta(top(), start()); }
    size_t used_bytes() const                      { return pointer_delta(top(), start(), 1); }
    size_t free() const                            { return pointer_delta(end(), top()); }
    // Don't discard tlab if remaining space is larger than this.
    size_t refill_waste_limit() const              { return _refill_waste_limit; }

    // Allocate size HeapWords. The memory is NOT initialized to zero.
    inline HeapWord* allocate(size_t size);

    // Reserve space at the end of TLAB
    static size_t end_reserve() {
      int reserve_size = typeArrayOopDesc::header_size(T_INT);
      return MAX2(reserve_size, VM_Version::reserve_for_allocation_prefetch());
    }
    static size_t alignment_reserve()              { return align_object_size(end_reserve()); }
    static size_t alignment_reserve_in_bytes()     { return alignment_reserve() * HeapWordSize; }

    // Return tlab size or remaining space in eden such that the
    // space is large enough to hold obj_size and necessary fill space.
    // Otherwise return 0;
    inline size_t compute_size(size_t obj_size);

    // Record slow allocation
    inline void record_slow_allocation(size_t obj_size);

    // Initialization at startup
    static void startup_initialization();

    // Make an in-use tlab parsable, optionally also retiring it.
    void make_parsable(bool retire);

    // Retire in-use tlab before allocation of a new tlab
    void clear_before_allocation();

    // Accumulate statistics across all tlabs before gc
    static void accumulate_statistics_before_gc();

    // Resize tlabs for all threads
    static void resize_all_tlabs();

    void fill(HeapWord* start, HeapWord* top, size_t new_size);
    void initialize();

    static size_t refill_waste_limit_increment()   { return TLABWasteIncrement; }

    // Code generation support
    static ByteSize start_offset()                 { return byte_offset_of(ThreadLocalAllocBuffer, _start); }
    static ByteSize end_offset()                   { return byte_offset_of(ThreadLocalAllocBuffer, _end  ); }
    static ByteSize top_offset()                   { return byte_offset_of(ThreadLocalAllocBuffer, _top  ); }
    static ByteSize pf_top_offset()                { return byte_offset_of(ThreadLocalAllocBuffer, _pf_top  ); }
    static ByteSize size_offset()                  { return byte_offset_of(ThreadLocalAllocBuffer, _desired_size ); }
    static ByteSize refill_waste_limit_offset()    { return byte_offset_of(ThreadLocalAllocBuffer, _refill_waste_limit ); }

    static ByteSize number_of_refills_offset()     { return byte_offset_of(ThreadLocalAllocBuffer, _number_of_refills ); }
    static ByteSize fast_refill_waste_offset()     { return byte_offset_of(ThreadLocalAllocBuffer, _fast_refill_waste ); }
    static ByteSize slow_allocations_offset()      { return byte_offset_of(ThreadLocalAllocBuffer, _slow_allocations ); }

    void verify();
  };
  ```
3. 
