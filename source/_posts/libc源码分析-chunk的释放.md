---
title: libc源码分析-chunk的释放
date: 2019-03-10 15:27:25
tags:
---

### _libc_free
```C
__libc_free (void *mem)
{
  mstate ar_ptr;
  mchunkptr p;                          /* chunk corresponding to mem */
  void (*hook) (void *, const void *)
    = atomic_forced_read (__free_hook);
  if (__builtin_expect (hook != NULL, 0))
    {
      (*hook)(mem, RETURN_ADDRESS (0));
      return;
    }
  if (mem == 0)                              /* free(0) has no effect */
    return;
  p = mem2chunk (mem);
  if (chunk_is_mmapped (p))                       /* release mmapped memory. */
    {
      /* See if the dynamic brk/mmap threshold needs adjusting.
         Dumped fake mmapped chunks do not affect the threshold.  */
      if (!mp_.no_dyn_threshold
          && chunksize_nomask (p) > mp_.mmap_threshold
          && chunksize_nomask (p) <= DEFAULT_MMAP_THRESHOLD_MAX
          && !DUMPED_MAIN_ARENA_CHUNK (p))
        {
          mp_.mmap_threshold = chunksize (p);
          mp_.trim_threshold = 2 * mp_.mmap_threshold;
          LIBC_PROBE (memory_mallopt_free_dyn_thresholds, 2,
                      mp_.mmap_threshold, mp_.trim_threshold);
        }
      munmap_chunk (p);
      return;
    }

  MAYBE_INIT_TCACHE ();
  ar_ptr = arena_for_chunk (p);
  _int_free (ar_ptr, p, 0);
}
```

## _int_free

```C
/*
   ------------------------------ free ------------------------------
 */
static void
_int_free (mstate av, mchunkptr p, int have_lock)
{
  INTERNAL_SIZE_T size;        /* its size */
  mfastbinptr *fb;             /* associated fastbin */
  mchunkptr nextchunk;         /* next contiguous chunk */
  INTERNAL_SIZE_T nextsize;    /* its size */
  int nextinuse;               /* true if nextchunk is used */
  INTERNAL_SIZE_T prevsize;    /* size of previous contiguous chunk */
  mchunkptr bck;               /* misc temp for linking */
  mchunkptr fwd;               /* misc temp for linking */
  size = chunksize (p);
  /* Little security check which won't hurt performance: the
     allocator never wrapps around at the end of the address space.
     Therefore we can exclude some size values which might appear
     here by accident or by "design" from some intruder.  */
  if (__builtin_expect ((uintptr_t) p > (uintptr_t) -size, 0)
      || __builtin_expect (misaligned_chunk (p), 0))
    malloc_printerr ("free(): invalid pointer");

  /* We know that each chunk is at least MINSIZE bytes in size or a
     multiple of MALLOC_ALIGNMENT.  */
  /*检查chunk的size是否满足大小且对齐*/
  if (__glibc_unlikely (size < MINSIZE || !aligned_OK (size)))
    malloc_printerr ("free(): invalid size");

  /*检查下个chunk的PREV_INUSE(lsb)位,判断当前chunk是否被使用,但是在释放fastchunk时,该位会被忽略*/  
  check_inuse_chunk(av, p);
#if USE_TCACHE
  {
    size_t tc_idx = csize2tidx (size);
    if (tcache
        && tc_idx < mp_.tcache_bins
        && tcache->counts[tc_idx] < mp_.tcache_count)
      {
        tcache_put (p, tc_idx);
        return;
      }
  }
#endif

  /*
    If eligible, place chunk on a fastbin so it can be found
    and used quickly in malloc.
    判断chunk的size是否满足fastchunk的大小(64 bytes on x32, 128 bytes on x64)
  */
  if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())
#if TRIM_FASTBINS
      /*
        If TRIM_FASTBINS set, don't place chunks
        bordering top into fastbins
        该变量默认为0
      */
      && (chunk_at_offset(p, size) != av->top)
#endif
      ) {
    /*检查next chunk的size是否大于2倍的SIZE_SZ(SIZE_SZ=8 on x64)*/
    if (__builtin_expect (chunksize_nomask (chunk_at_offset (p, size))
                          <= 2 * SIZE_SZ, 0)
    /*检查next chunk的size是否大于av->system_mem(默认情况下av->system_mem=128kb即0x21000 bytes)*/                      
        || __builtin_expect (chunksize (chunk_at_offset (p, size))
                             >= av->system_mem, 0))
      {
        bool fail = true;
        /* We might not have a lock at this point and concurrent modifications
           of system_mem might result in a false positive.  Redo the test after
           getting the lock.  */
        if (!have_lock)
          {
            __libc_lock_lock (av->mutex);
            fail = (chunksize_nomask (chunk_at_offset (p, size)) <= 2 * SIZE_SZ
                    || chunksize (chunk_at_offset (p, size)) >= av->system_mem);
            __libc_lock_unlock (av->mutex);
          }
        if (fail)
          malloc_printerr ("free(): invalid next size (fast)");
      }
    free_perturb (chunk2mem(p), size - 2 * SIZE_SZ);
    atomic_store_relaxed (&av->have_fastchunks, true);
    unsigned int idx = fastbin_index(size);
    fb = &fastbin (av, idx);
    /* Atomically link P to its fastbin: P->FD = *FB; *FB = P;  */
    mchunkptr old = fastbin_push_entry (fb, p);
    /* Check that size of fastbin chunk at the top is the same as
       size of the chunk that we are adding.  We can dereference OLD
       only if we have the lock, otherwise it might have already been
       allocated again.  */
    if (have_lock && old != NULL
        && __builtin_expect (fastbin_index (chunksize (old)) != idx, 0))
      malloc_printerr ("invalid fastbin entry (free)");
  }
  /*
    Consolidate other non-mmapped chunks as they arrive.
  */
  else if (!chunk_is_mmapped(p)) {
    /* If we're single-threaded, don't lock the arena.  */
    if (SINGLE_THREAD_P)
      have_lock = true;
    if (!have_lock)
      __libc_lock_lock (av->mutex);
    nextchunk = chunk_at_offset(p, size);
    /* Lightweight tests: check whether the block is already the
       top block.  */
    if (__glibc_unlikely (p == av->top))
      malloc_printerr ("double free or corruption (top)");
    /* Or whether the next chunk is beyond the boundaries of the arena.  */
    if (__builtin_expect (contiguous (av)
                          && (char *) nextchunk
                          >= ((char *) av->top + chunksize(av->top)), 0))
        malloc_printerr ("double free or corruption (out)");
    /* Or whether the block is actually not marked used.  */
    if (__glibc_unlikely (!prev_inuse(nextchunk)))
      malloc_printerr ("double free or corruption (!prev)");
    nextsize = chunksize(nextchunk);
    if (__builtin_expect (chunksize_nomask (nextchunk) <= 2 * SIZE_SZ, 0)
        || __builtin_expect (nextsize >= av->system_mem, 0))
      malloc_printerr ("free(): invalid next size (normal)");
    free_perturb (chunk2mem(p), size - 2 * SIZE_SZ);
    /* consolidate backward */
    if (!prev_inuse(p)) {
      prevsize = prev_size (p);
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize));
      if (__glibc_unlikely (chunksize(p) != prevsize))
        malloc_printerr ("corrupted size vs. prev_size while consolidating");
      unlink_chunk (av, p);
    }
    if (nextchunk != av->top) {
      /* get and clear inuse bit */
      nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
      /* consolidate forward */
      if (!nextinuse) {
        unlink_chunk (av, nextchunk);
        size += nextsize;
      } else
        clear_inuse_bit_at_offset(nextchunk, 0);
      /*
        Place the chunk in unsorted chunk list. Chunks are
        not placed into regular bins until after they have
        been given one chance to be used in malloc.
      */
      bck = unsorted_chunks(av);
      fwd = bck->fd;
      if (__glibc_unlikely (fwd->bk != bck))
        malloc_printerr ("free(): corrupted unsorted chunks");
      p->fd = fwd;
      p->bk = bck;
      if (!in_smallbin_range(size))
        {
          p->fd_nextsize = NULL;
          p->bk_nextsize = NULL;
        }
      bck->fd = p;
      fwd->bk = p;
      set_head(p, size | PREV_INUSE);
      set_foot(p, size);
      check_free_chunk(av, p);
    }
    /*
      If the chunk borders the current high end of memory,
      consolidate into top
    */
    else {
      size += nextsize;
      set_head(p, size | PREV_INUSE);
      av->top = p;
      check_chunk(av, p);
    }
    /*
      If freeing a large space, consolidate possibly-surrounding
      chunks. Then, if the total unused topmost memory exceeds trim
      threshold, ask malloc_trim to reduce top.
      Unless max_fast is 0, we don't know if there are fastbins
      bordering top, so we cannot tell for sure whether threshold
      has been reached unless fastbins are consolidated.  But we
      don't want to consolidate on each free.  As a compromise,
      consolidation is performed if FASTBIN_CONSOLIDATION_THRESHOLD
      is reached.
    */
    if ((unsigned long)(size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
      if (atomic_load_relaxed (&av->have_fastchunks))
        malloc_consolidate(av);
      if (av == &main_arena) {
#ifndef MORECORE_CANNOT_TRIM
        if ((unsigned long)(chunksize(av->top)) >=
            (unsigned long)(mp_.trim_threshold))
          systrim(mp_.top_pad, av);
#endif
      } else {
        /* Always try heap_trim(), even if the top chunk is not
           large, because the corresponding heap might go away.  */
        heap_info *heap = heap_for_ptr(top(av));
        assert(heap->ar_ptr == av);
        heap_trim(heap, mp_.top_pad);
      }
    }
    if (!have_lock)
      __libc_lock_unlock (av->mutex);
  }
  /*
    If the chunk was allocated via mmap, release via munmap().
  */
  else {
    munmap_chunk (p);
  }
}
```

_int_free中有几个重要的if判断
####1. `if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())`
这个if和fastbin有关，用来判断要释放的chunk是否是fast chunk，关于MAX_FAST_SIZE的定义如下：
```C
/* The maximum fastbin request size we support */
#define MAX_FAST_SIZE     (80 * SIZE_SZ / 4)
```
SIZE_SZ=8 on x64，SIZE_SZ=4 on x32，这个MAX_FAST_SIZE是包括chunk头的大小，所以64位下MAX_FAST_SIZE=160 bytes,32位下MAX_FAST_SIZE=80 bytes，但是实际使用过程中，64位下最大为128bytes，32位下最大为64bytes
