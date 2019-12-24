---
layout: post
title: Writing a seqlock in Rust
---

A seqlock — or "sequence lock" — is an optimized implementation of a reader-writer lock. In a seqlock "the data can be 'protected' by a sequence number. The sequence number starts at zero, and is incremented before and after writing the object. Each reader checks the sequence number before and after reading. If both values are the same and even, then there cannot have been any concurrent increments, and the reader must have seen consistent data."

## Initial implementation

Let's start with a skeleton of the implementation. This version has multiple problems and room for improvements, which we are going to explore next.

```rust
use std::cell::UnsafeCell;
use std::sync::atomic::{AtomicUsize, Ordering};

pub struct SeqLock<T: Copy> {
    seq: AtomicUsize,
    data: UnsafeCell<T>,
}

unsafe impl<T: Copy + Send> Send for SeqLock<T> {}
unsafe impl<T: Copy + Send> Sync for SeqLock<T> {}

impl<T: Copy> SeqLock<T> {
    pub fn new(val: T) -> SeqLock<T> {
        SeqLock {
            seq: AtomicUsize::new(0),
            data: UnsafeCell::new(val),
        }
    }

    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 { continue }
            let result = unsafe { *self.data.get() };
            let seq2 = self.seq.load(Ordering::Relaxed);
            if seq1 == seq2 {
                return result;
            }
        }
    }

    pub fn write(&self, new: T) {
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 { continue }
            if seq1 != self.seq.compare_and_swap(seq1,
                                                 seq1.wrapping_add(1),
                                                 Ordering::Relaxed)
            {
                continue;
            }
            unsafe { *self.data.get() = new; }
            self.seq.store(seq1.wrapping_add(2), Ordering::Relaxed);
            break;
        }
    }
}
```

## Reads and writes

`write` can write to `data` while other threads are concurrently reading from `data`. It is invalid to do this using shared `&` references and mutable `&mut` references, because that would violate the aliasing guarantees. We may never use a `&mut` reference when it is not the only unique reference at that point. So we have to be careful to make sure all access to `data` happen through raw pointers.

In the initial example we read and write to `data` by doing a simple assignment. Would it make sense to use [`ptr::read`](https://doc.rust-lang.org/std/ptr/fn.read.html) and [`ptr::write`](https://doc.rust-lang.org/std/ptr/fn.write.html)? Simple assignment works because we have a `Copy` bound on `T`. For a seqlock such a bound is necessary, any value that is not trivially copyable is pretty much guaranteed to cause problems. `ptr::read` and `ptr::write` allow use to use any type, without requiring `Copy`. So they are more powerful, but not necessary for us.

Reading from `data` while another thread is writing to it is a data race. The data may be inconsistent, values may be partially written ("teared") or even stale because the contents in the cache of one processor core has not reached the other yet. We check the sequence number after reading `data` and start over if we detect there was a write in progress. Still, is there something we should worry about?

The reads from `data` are sandwiched between two atomic accesses, and so are writes to `data`. If we get the atomic orderings right in the next sections, they should be enough to ensure our reads and writes stay between them. Then we don't really have to be afraid of the compiler or processor inserting speculative reads or writes. Yet it seems good to inform the compiler about our intention and use volatile accesses.

The documentation of [`ptr::read_volatile`](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html) and [`ptr::write_volatile`](https://doc.rust-lang.org/std/ptr/fn.read_volatile.html) comes with the warning:
> Rust does not currently have a rigorously and formally defined memory model, so the precise semantics of what "volatile" means here is subject to change over time. That being said, the semantics will almost always end up pretty similar to C11's definition of volatile.

Great. So we are in somewhat uncharted territory. A thread on [Rust Internals](https://internals.rust-lang.org/t/volatile-and-sensitive-memory) and one in the [unsafe-code-guidelines](https://github.com/rust-lang/unsafe-code-guidelines/issues/33) have some discussion on how to use volatile correctly. Basically it only make sense when used with memory-mapped IO. Some of the things we seem to have to keep in mind:
- Never create references, because they come with the property they are dereferencable. LLVM is then allowed to insert speculative reads and writes. Always use raw pointers.
- Never access it with a non-volatile operation, they would act as a hint to the compiler that volatile operations are not really necessary. Always use `ptr::read_volatile` and `ptr::write_volatile`.

An alternative may be [atomic memcpy](https://github.com/rust-lang/rust/issues/58599), although that is not available in Rust yet and uses the somewhat questionable LLVM-specific `Unordered` atomic ordering. We'll use volatile accesses for now because they describe our intention resonably well, and that is all that matters because we care less about the guarantees volatile should provide:

```rust
    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 { continue }
            let result = unsafe { ptr::read_volatile(self.data.get()) };
            // ^-- CHANGED
            let seq2 = self.seq.load(Ordering::Relaxed);
            if seq1 == seq2 {
                return result;
            }
        }
    }

    pub fn write(&self, new: T) {
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 { continue }
            if seq1 != self.seq.compare_and_swap(seq1,
                                                 seq1.wrapping_add(1),
                                                 Ordering::Relaxed)
            {
                continue;
            }
            unsafe { ptr::write_volatile(self.data.get(), new); } // CHANGED
            self.seq.store(seq1.wrapping_add(2), Ordering::Relaxed);
            break;
        }
    }
```


## Release and acquire

Release and acquire operations on one atomic are the standard way to synchronize data between threads. Thread A does regular writes to some data and finishes with a release store on an atomic. Thread B does an acquire read on the same atomic, the processor makes sure all writes by Thread A become visible to Thread B, and it can do normal reads on the data.

In our seqlock implementation we need to do the same: the final store in `write` must use `Release` ordering, and the first read (to be precise: the first read of an even sequence number) in `read` must use `Acquire` ordering.

```rust
    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Acquire); // CHANGED
            if seq1 & 1 != 0 { continue }
            let result = unsafe { ptr::read_volatile(self.data.get()) };
            let seq2 = self.seq.load(Ordering::Relaxed);
            if seq1 == seq2 {
                return result;
            }
        }
    }

    pub fn write(&self, new: T){
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 { continue }
            if seq1 != self.seq.compare_and_swap(seq1,
                                                 seq1.wrapping_add(1),
                                                 Ordering::Relaxed)
            {
                continue;
            }
            unsafe { ptr::write_volatile(self.data.get(), new); }
            self.seq.store(seq1.wrapping_add(2), Ordering::Release); // CHANGED
            break;
        }
    }
```

Should we also do an acquire before writing to `data`? Strictly speaking not, because we don't care about the old contents. It will just be overwritten. But as we will see next, it is necessary.

## Instruction reordering

This is the subtle problem with atomic operations dat is not called out often enough in my opinion. Processors can execute reads and/or writes in a different order than how you wrote them. Reads may be prefetched, writes may be cached, the processor can reorder things as it sees fit. It may execute instructions a thousand times in one order, and then once in another, giving hard to debug issues.

With atomics you have to be very careful to state in which order instructions are to be executed. A few rules:
* _Regular reads and writes_ may be freely reordered. Only reads and writes to the _same value_ will happen in order.
* _Relaxed_ atomic loads and stores may be freely reordered. Only loads and stores to the _same atomic_ will happen in order.
* An atomic _Acquire load_ will prevent any reads and writes that should happen after it from being reordered to before the acquire. This guarantees that any reads and writes on shared data (typically inside an `UnsafeCell` in Rust) will operate on the most recently acquired version. Just like regular reads and writes, loads and stores on other atomics can also not be reordered to happen before the acquire.
* An atomic _Release store_ will prevent any reads and writes that should happen before it from being reordered to after the release. This guarantees that all operations on shared data are finished before releasing the data to other threads. In the same way loads and stores on other atomics are guaranteed to not get reordered to after the release.

This is our basic toolbox. `Relaxed` doesn't say anything about ordering, `Acquire` keeps things after it, and `Release` keeps things before it.

A seqlock is very sensitive to the order of instructions. The read first has to check the sequence number, then do the volatile read, and finish with another check of the sequence number. A write first has to update the sequence number, then do the write, and finish with updating the sequence number.

We'll start with `write`. The update of `seq` in `compare_and_swap` must happen before we do any writes to `data`. How can we establish an order that the processor will respect? We either have to do the `compare_and_swap` with `Acquire` ordering, or do the following writes with `Release` ordering. But the following writes are volatile (regular) writes, so we have to add the ordering to the `compare_and_swap`.

Next the final store to `seq` in `write`. It must happen after the writes to `data`, and that is exactly what the `Release` ordering that is already there guarantees. 

```rust
    pub fn write(&self, new: T) {
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 { continue }
            if seq1 != self.seq.compare_and_swap(seq1,
                                                 seq1.wrapping_add(1),
                                                 Ordering::Acquire) // CHANGED
            {
                continue;
            }
            unsafe { ptr::write_volatile(self.data.get(), new); }
            self.seq.store(seq1.wrapping_add(2), Ordering::Release);
            break;
        }
    }
```

Now the same for `read`. The initial `Acquire` load ensures the next reads stay after it. How do we make sure the final load on `self.seq` happens after the reads? What we want there is a `Release` load, but that is something that doesn't exist. This is a pretty uncommon synchronization problem, that has no easy solution. Seqlocks just don't fit well in the C++ memory model. The paper [Can Seqlocks Get Along With Programming Language Memory Models?](https://www.hpl.hp.com/techreports/2012/HPL-2012-68.pdf) discusses some options. One options is using a 'read-dont-modify-write' operation with `Release` ordering:

```rust
    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Acquire);
            if seq1 & 1 != 0 { continue }
            let result = unsafe { ptr::read_volatile(self.data.get()) };
            let seq2 = self.seq.compare_and_swap(seq1, seq1, Ordering::Release);
            // ^-- CHANGED
            if seq1 == seq2 {
                return result;
            }
        }
    }
```

As the paper says, this "has the huge disadvantage that the final load from `seq` has been turned into an operation that normally requires exclusive access to the cache line. This reintroduces coherence misses as ownership of the cache line is transferred between threads executing read-only critical sections." On other words, performance plummets by an order of magnitude.

Another options is to do atomic reads on the data instead of volatile reads. We can than make every read do an `Acquire`. Or we could add an `Acquire` fence after the last atomic read on `data`. Fences are [very tricky](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence), but you can imagine it as telling the processor to turn the last atomic load it executed into an acquire load. (Note: a `Release` fence works in the opposite direction, it binds to the next atomic store).

So we are somewhat stuck without a good solution. The 'read-don't-modify-write' options seems like the safest bad choice for now. It is possible to write processor specific code that uses barriers to achieve what we want, something to explore later.

## Spin loop

Both `read` and `write` keep looping if `seq1` is not even. It is not polite to keep claiming the CPU waiting like that. We have three options, depending on how long we expect to wait:
- [`std::sync::atomic::spin_loop_hint()`](https://doc.rust-lang.org/std/sync/atomic/fn.spin_loop_hint.html) for short waits. This is a hint to the processor, which allows it to "optimize its behavior by, for example, saving power or switching hyper-threads."
- [`std::thread::yield_now()`](https://doc.rust-lang.org/std/thread/fn.yield_now.html) signals the OS scheduler it will have nothing to do for some time, so it can run some other thread or process. For longer waits.
- A synchronization primitive like Mutex, or even better, Futex. The mutex is locked while one thread is writing, and other threads on wait on it.

For seqlock the waits are expected to be short, `spin_loop_hint` is the best choice. It has to be given on every loop iteration:

```rust
    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Acquire);
            if seq1 & 1 != 0 {
                spin_loop_hint();
                continue;
            }
            let result = unsafe { ptr::read_volatile(self.data.get()) };
            let seq2 = self.seq.compare_and_swap(seq1, seq1, Ordering::Release);
            // ^-- CHANGED
            if seq1 == seq2 {
                return result;
            }
        }
    }

    pub fn write(&self, new: T) {
        loop {
            let seq1 = self.seq.load(Ordering::Relaxed);
            if seq1 & 1 != 0 {
                spin_loop_hint();
                continue;
            }
            if seq1 != self.seq.compare_and_swap(seq1,
                                                 seq1.wrapping_add(1),
                                                 Ordering::Acquire)
            {
                continue;
            }
            unsafe { ptr::write_volatile(self.data.get(), new); }
            self.seq.store(seq1.wrapping_add(2), Ordering::Release);
            break;
        }
    }
```

## Weak compare_exchange

It is worth optimizing our implementation? Probably not, but it's just fun doing.

`compare_and_swap` is actually a simplified version of [`compare_exchange`](https://doc.rust-lang.org/core/sync/atomic/struct.AtomicUsize.html#method.compare_exchange). `compare_exchange` can take two orderings, one for the successful case, and one to use on failure. This makes it a bit more tricky, table:

| ordering             | load (success) | load (failure) | store (success)
|----------------------|----------------|----------------|-----------------
| `Relaxed`, `Relaxed` | `Relaxed`      | `Relaxed`      | `Relaxed`
| `Acquire`, `Acquire` | `Acquire`      | `Acquire`      | `Relaxed`
| `Acquire`, `Relaxed` | `Acquire`      | `Relaxed`      | `Relaxed`
| `Release`, `Relaxed` | `Relaxed`      | `Relaxed`      | `Release`
| `AcqRel`,  `Acquire` | `Acquire`      | `Acquire`      | `Release`
| `AcqRel`,  `Relaxed` | `Acquire`      | `Relaxed`      | `Release`

In `write` we don't really care about the acquire ordering in the case the CAS fails, because it just continues from the start of the loop again. We might as well use `compare_exchange` with an (`Acquire`, `Relaxed`) pair. Who knows if it helps a little on some architecture? 

`compare_exchange` also sets a flag to indicate if the CAS was successful, we don't even have to check the returned integer. In Rust this is encoded with a `Result` type. There also is `compare_exchange_weak`, which may spuriously fail, even when the returned integer indicates it could have succeeded. This will yield better performance on some platforms. The common advise about when to use `compare_exchange_weak` is: if you would have to introduce a loop only because of spurious failure, don't use it. But if you have a loop anyway, `weak` is the better choice.

In `write` we are doing a CAS in a loop, so lets use `compare_exchange_weak`. The CAS also gives us the old value of `seq` on failure, so why would we do another load of `seq` again on the next loop iteration? We may as wel re-use it and move the first load out of the loop.

```rust
    pub fn write(&self, new: T) {
        let mut seq1 = self.seq.load(Ordering::Relaxed);
        loop {
            if seq1 & 1 != 0 {
                spin_loop_hint();
                continue;
            }
            if let Err(x) = self.seq.compare_exchange_weak(seq1,
                                                           seq1.wrapping_add(1),
                                                           Ordering::Acquire,
                                                           Ordering::Relaxed)
            {
                seq1 = x;
                continue;
            }
            *self.data.get() = new;
            self.seq.store(seq1.wrapping_add(2), Ordering::Release);
            break;
        }
    }
```

In `read` we are also doing a CAS inside a loop. But you can't call that 'read-don't-modify-write'-trick a CAS-loop, it is there to prevent reordering. I am not sure if using `compare_exchange_weak` is wise in this case. Would it still ensure the correct ordering in the failure case, a case which normally can't exist?

Can we do the other optimization in `read`, to not do the initial load on `seq` on every loop iteration? The initial load is done with `Acquire` ordering. If we wan't to do this optimization, the second load (in `seq2`) must also be done with `Acquire` ordering. That would pessimize the happy path, so it is best to leave `read` as it is.

## Going architecture-specific

Within the memory model the `read` method of our seqlock remains far from optimal. What we want is a barrier between de volatile reads on the second atomic load of the sequence counter. Officially an acquire fence is not usable, because it needs a preceding atomic load. But it might just compile to what we need. Or we can use platform-specific intrinsics to get the barrier we need.

On x86 the processor already has strong enough orderings. All we need is to stop the compiler from reordering our loads using [`std::sync::atomic::compiler_fence`](https://doc.rust-lang.org/std/sync/atomic/fn.compiler_fence.html). It's argument can be either `Release` of `Acquire` (or both, `AcqRel`). It doesn't really matter if we choose to prevent the preceding load from moving to after the fence, or the following load from moving to before it.

On ARM and AARCH-64 we would need a `dmb` instruction, a _data memory barrier_. It's argument should be `ISHLD`: a load-load barrier over the inner sharable domain, which typically means over the same processor (with multiple cores). The intrinsic is available in nightly's standard library in [`core::arch::arm::_dmb`](https://doc.rust-lang.org/core/arch/arm/fn.__dmb.html), although the argument we have to supply seem to be missing.

On Power architectures we need an `lwsync` instruction. A _light-weight sync_ is restricted to system memory (instead of also forming a barrier with device memory, something we don't need). It seems to not yet be exposed as an intrinsic in the standard library.

I am not sure wat the status is with barriers and MIPS.

Again we are stuck without a great solution. The intrinsics we need are not available, or nightly-only. But now that we know what we need, what does a lonely `fence(Ordering::Acquire)` compile down to? It would not be correct, but we are going the route of architecture-specific code now. If it generates the code we need, that is ok for now. Let's test:

| architecture                            | instruction
|-----------------------------------------|--------------------------------
| [ARM](https://godbolt.org/z/kaB4Lb)     | `mcr p15, #0, r12, c7, c10, #5`
| [AARCH64](https://godbolt.org/z/YxPobk) | `dmb ishld`
| [PowerPC](https://godbolt.org/z/PfC4ze) | `lwsync`
| [mips](https://godbolt.org/z/AnPDCo)    | `sync`

x86 is not included, because we know it would even function without the barrier (but we would still need to insert a compiler fence). AARCH64, PowerPC generate the code we want. Mips seems plausable. What's up with ARM? According to [this table](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0344k/BCGFFBFD.html) this instructions forms a Data Memory Barrier, so appearently ok also.

Okay, then let's use `fence(Ordering::Acquire)`, add a big comment about which compiler version we tested to work, and make it only compile for the processors we verified.

```rust
    // IMPORTANT: Our use of a fence is not correct according to the C++ memory
    // model, because it is not preceded by an atomic load.
    // For the following processors it generates the correct assembly, i.e.
    // inserts a load barrier, with Rust version 1.39; 2019-12-23.
    #[cfg(any(target_arch="x86", target_arch="x86_64",
              target_arch="arm", target_arch="armv7",
              target_arch="aarch64",
              target_arch="powerpc",
              target_arch="mips"))]
    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Acquire);
            if seq1 & 1 != 0 {
                spin_loop_hint();
                continue;
            }
            let result = unsafe { ptr::read_volatile(self.data.get()) };
            fence(Ordering::Acquire);
            let seq2 = self.seq.load(Ordering::Relaxed);
            if seq1 == seq2 {
                return result;
            }
        }
    }
```

## Final result

```rust
use std::cell::UnsafeCell;
use std::ptr;
use std::sync::atomic::{AtomicUsize, fence, Ordering, spin_loop_hint};

pub struct SeqLock<T: Copy> {
    seq: AtomicUsize,
    data: UnsafeCell<T>,
}

unsafe impl<T: Copy + Send> Send for SeqLock<T> {}
unsafe impl<T: Copy + Send> Sync for SeqLock<T> {}

impl<T: Copy> SeqLock<T> {
    pub fn new(val: T) -> SeqLock<T> {
        SeqLock {
            seq: AtomicUsize::new(0),
            data: UnsafeCell::new(val),
        }
    }

    // IMPORTANT: Our use of a fence is not correct according to the C++ memory
    // model, because it is not preceded by an atomic load.
    // For the following processors it generates the correct assembly, i.e.
    // inserts a load barrier, with Rust version 1.39; 2019-12-23.
    #[cfg(any(target_arch="x86", target_arch="x86_64",
              target_arch="arm", target_arch="armv7",
              target_arch="aarch64",
              target_arch="powerpc",
              target_arch="mips"))]
    pub fn read(&self) -> T {
        loop {
            let seq1 = self.seq.load(Ordering::Acquire);
            if seq1 & 1 != 0 {
                spin_loop_hint();
                continue;
            }
            let result = unsafe { ptr::read_volatile(self.data.get()) };
            fence(Ordering::Acquire);
            let seq2 = self.seq.load(Ordering::Relaxed);
            if seq1 == seq2 {
                return result;
            }
        }
    }

    pub fn write(&self, new: T) {
        let mut seq1 = self.seq.load(Ordering::Relaxed);
        loop {
            if seq1 & 1 != 0 {
                spin_loop_hint();
                continue;
            }
            if let Err(x) = self.seq.compare_exchange_weak(seq1,
                                                           seq1.wrapping_add(1),
                                                           Ordering::Acquire,
                                                           Ordering::Relaxed)
            {
                seq1 = x;
                continue;
            }
            *self.data.get() = new;
            self.seq.store(seq1.wrapping_add(2), Ordering::Release);
            break;
        }
    }
}

```

I probably made plenty of mistakes. But this exploring is fun. I love to read the educational deep-dive posts about Rust (there seemed to be more of those around the Rust 1.0 time), and wished there were more about writing unsafe code and concurrency. This is my small contribution. Please comment on whatever can be improved!
