---
layout: post
title: Interior mutability patterns
---

Rusts type system requires that there only ever is one mutable reference to a value _or_ one or more shared references. What happens when you need multiple references to some value, but also need to mutate through them? We use a trick called _interor mutability_: to the outside world you act like a value is immutable so multiple references are allowed. But internally the type is actually mutable.

All types that provide interior mutability have an `UnsafeCell` at their core. `UnsafeCell` is the only primitive that allows multiple mutable pointers to its interior, without violating aliasing rules. The only way to use it safely is to only mutate the wrapped value when there are no other readers. No, the garantee has to be even stronger: _we can not mutate it and can not create a mutable reference to the wrapped value while there are shared references to its value._

Both [the book](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) and the [`std::cell` module](https://doc.rust-lang.org/std/cell/index.html) give a good alternative explanation of interor mutability.

- What are some patterns that have been developed to use interior mutability safely?
- How do multithreaded synchronization primitives that provide interior mutability follow similar principles?

<!--more-->

We are going to look at four patterns:
- [Never handing out references](#never-handing-out-references)
- [Ensuring at runtime a mutable reference is unique](#ensuring-at-runtime-a-mutable-reference-is-unique)
- [Mutability only during initialization](#mutability-only-during-initialization)
- [Using a seperate 'owner' or 'token' to track references](#using-a-seperate-owner-or-token-to-track-references)

## Never handing out references

One way to make sure there are no shared references when mutating the wrapped value is to never hand out any references at all. The [`Cell`](https://doc.rust-lang.org/std/cell/struct.Cell.html) type in the standard library is the prime example of this pattern.

### Cell
Basic API ([see documentation](https://doc.rust-lang.org/std/cell/struct.Cell.html)):
```rust
impl<T> Cell<T> {
    pub fn set(&self, val: T)
    pub fn replace(&self, val: T) -> T
}
impl<T: Copy> Cell<T> {
    pub fn get(&self) -> T
}
impl<T: Default> Cell<T> {
    pub fn take(&self) -> T
}
```

The idea of `Cell` is that when reading its contents you always either `set` or `get` the entire contents of the `Cell`. With a single thread there can only ever be one such operation in progress, so there is no change of multiple mutable accesses: a safe API for interior mutability.

Before Rust 1.17 ([RFC](https://rust-lang.github.io/rfcs/1651-movecell.html)) `Cell` was only implemented for `Copy` types with the `set` and `get` methods. Crates like [movecell](https://docs.rs/movecell/0.2/movecell/struct.MoveCell.html) and [mitochondria](https://docs.rs/mitochondria/1/mitochondria/struct.MoveCell.html) demonstrated a possible extension: `MoveCell`. You can use `Cell` with non-`Copy` types by taking its value out when reading, and replacing it with something else.

That brings us to the flexible API above: `get` is the normal way to read the value of a `Cell`, but is only available for `Copy` types. If a type can't be copied, you can always `replace` it with some other value. And if the type comes with a `Default`, you can `take` out the value and replace it with the default.

`Cell` does not work with multiple threads. It has no way to coordinate with another thread that `set` is not used while the other uses `get`. Enter the synchronization primitives for a `Cell`-like API below.

### Atomics
Did you know that Rusts [atomic integers](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicU32.html) are just an `UnsafeCell` around an integer?
```rust
#[repr(C, align(4))]
pub struct AtomicU32 {
    v: UnsafeCell<u32>,
}
```
With atomic operations the processor ensures any read or write will be synchronized across threads. What can be done for integers, can be done for any type that is small enough to fit in an integer. Usualy this means the types can be up to 8 bytes, and most 64-bit platforms support atomic operations on up to 16 bytes.

Multiple crates allow using an `UnsafeCell` with atomic operations on various small types:
- [Crossbeam](https://crates.io/crates/crossbeam) with its `AtomicCell<T>`
- [`atomic`](https://crates.io/crates/atomic) with `Atomic<T>` (requires types are `Copy`).
- [`atomig`](https://crates.io/crates/atomig), with `Atomic<T>` (requires a `#[derive(atom)]`).

If you look past the names of the methods on `AtomicCell`, just notice how close the atomic operations resemble the methods on `Cell`. Basic API ([see documentation](https://docs.rs/crossbeam/0.7/crossbeam/atomic/struct.AtomicCell.html)):
```rust
impl<T> AtomicCell<T> {
    pub fn store(&self, val: T)
    pub fn swap(&self, val: T) -> T
}
impl<T: Copy> AtomicCell<T> {
    pub fn load(&self) -> T
}
impl<T: Default> AtomicCell<T> {
    pub fn take(&self) -> T
}
impl<T: Copy + Eq> AtomicCell<T> {
    pub fn compare_and_swap(&self, current: T, new: T) -> T
    pub fn compare_exchange(&self, current: T, new: T) -> Result<T, T>
}
unsafe impl<T: Send> Sync for AtomicCell<T> {}
```

The methods `get`, `set` and `replace` are now `load`, `store` and `swap`. `AtomicCell` also offers `compare_and_swap`, one of the essential tools for concurrent algorithms. Currently this operation is [questionable](https://github.com/crossbeam-rs/crossbeam/issues/315), because transmuting small types with padding to an atomic integer and comparing it is probably Undefined Behavior. But I am confident the maintainers will figure out a solution.

In my opinion every of one of those crates has some disadvantage. Crossbeam has the potential problem with padding bytes. `atomic` has the same problem, and requires its types to be `Copy`. `atomig` can't be derived for types from other crates. Still, as long as you are careful to use it on small types without padding, you are good with all of them.

Atomic operations are definitely the best concurrent solution for interior mutability on this list.

### Locks
Exactly the same API as the one above for `AtomicCell` can be provided for larger types, by using some locking scheme.
- A `SeqLock` or _sequence lock_ protects its wrapped value with a sequence counter. When writing with `set` the sequence counter is incremented. Any read with `get` will first check the sequence counter, read the value, and check the sequence counter again. If the sequence counter changed, it will know there was a data race while reading and try again. *This is technically UB in Rust and therefore a questionable locking method.*
  The property that it does not have to do any atomic writes when just reading the value gives a very good performance. Note however that when a thread keeps writing to a seqlock, it can starve readers.
- A spinlock will set and clear a flag on every read or write. It doesn't allow for multiple concurrent reads.
- `atomic::Atomic<T>` does not include a lock with its type, but uses an global array of [64 spinlocks](https://github.com/Amanieu/atomic-rs/blob/v0.4.5/src/fallback.rs#L41). A hash of the address of the data is used as index to get a lock. This gives a nice little space efficiency.
- `crossbeam::AtomicCell<T>` does the same thing with a set of [97 global seqlocks](https://github.com/crossbeam-rs/crossbeam/blob/crossbeam-0.7.3/crossbeam-utils/src/atomic/atomic_cell.rs#L650). For every read it will make one attempt to use a seqlock as seqlock, and if that fails use it as spinlock.

With a concurrent `Cell`-like API it is best to use it like the initial version of `Cell`: only with types that are `Copy`. The trick to move non-`Copy` values in and out of a `Cell` with `replace` and `take` requires that you can always replace it with some other value that remains useful to other threads, which seems often impossible to me. For non-`Copy` types, use one of the reference based solutions below.

Note that with a `Cell`-like API locks are held for only a very short time, about as long as it takes to do a `memcpy`. So you usually don't need to ask the OS for assistance to wait until a thread is done (like `Mutex`, `RwLock` and co. below).

There is currently no good crate that offers a standalone `SeqLock` (see also my previous post on [writing a seqlock in Rust](../Writing-a-seqlock-in-Rust/)). The [seqlock](https://crates.io/crates/seqlock) crate hands out a mutable reference while reads may be in progress, which violates the aliasing rules. I also don't think there is a crate that offers a spinlock with the API described here. But the fallback for larger types in `atomic::Atomic<T>` and especially in `crossbeam::AtomicCell<T>` are pretty cool.

## Ensuring at runtime a mutable reference is unique

It is easily possible to hand out references, we just have to keep track of the number of shared and/or mutable references that are currently handed out. [_Smart Pointers_](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) give us the tools to do just that.

We register that a reference is handed out when creating a smart pointer. The smart pointer has a custom `Drop` implementation that unregisters itself when it goes out of scope. With a [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html) implementation it acts just like a regular shared reference. And with a [`DerefMut`](https://doc.rust-lang.org/std/ops/trait.DerefMut.html) it can also be used just like a mutable reference.

### RefCell
Basic API ([see documentation](https://doc.rust-lang.org/std/cell/struct.RefCell.html)):
```rust
impl<T: ?Sized> RefCell<T> {
    pub fn borrow(&self) -> Ref<T>
    pub fn borrow_mut(&self) -> RefMut<T>
}

impl<T: ?Sized> Deref for Ref<T> { /* ... */ }
impl<T: ?Sized> Drop for Ref<'_, T> { /* ... */ }

impl<T: ?Sized> DerefMut for RefMut<T> { /* ... */ }
impl<T: ?Sized> Deref for RefMut<T> { /* ... */ }
impl<T: ?Sized> Drop for RefMut<'_, T> { /* ... */ }
```

A `RefCell` makes use of a simple signed integer `borrow` to keep track of the number of borrows at runtime. Normal references are counted as positive numbers, and a mutable reference as -1. When you try to call `borrow_mut` while there are any open borrows, i.e. the integer is not 0, it panics: the essential ingredient to make this a safe API.

The methods on `RefCell` return the `Ref` and `RefMut` types. Those have a `DerefMut` and/or `Deref` implementation, though which the content of the `RefCell` can be accessed like they are a regular reference. When those 'smart references' are dropped, they decrement the reference count of the `RefCell` again.

The standard library also offers a `try_borrow` and `try_borrow_mut` method, which return an error instead of panicing the thread.

There currently seems to be Undefined Behavior in [some corner cases](https://github.com/rust-lang/rust/issues/63787) of `Ref` and `RefMut` because they internally convert a raw pointer to the value inside the `RefCell` too early to a normal reference. But we can be confident this is fixed in the standard library soon(ish).

### Mutex
Simplified API ([see documentation](https://doc.rust-lang.org/std/sync/struct.Mutex.html), the actual API is a bit more complex because it allows the mutex to be poisoned):
```rust
impl<T: ?Sized> Mutex<T> {
    pub fn lock(&self) -> MutexGuard<T>
}
unsafe impl<T: ?Sized + Send> Sync for Mutex<T> {}

impl<T: ?Sized> DerefMut for MutexGuard<T> { /* ... */ }
impl<T: ?Sized> Deref for MutexGuard<T> { /* ... */ }
impl<T: ?Sized> Drop for MutexGuard<'_, T> { /* ... */ }
```

A mutex provides _mutably exclusive_ access to its value. In a way this is synchronization primitive with only a limited part of `RefCell`s API: all borrows are mutable. Like `RefCell` all accesses happen through a smart pointer, in this case `MutexGuard`.

In modern implementations a mutex atomically sets a flag when the mutex gets locked, and unset it when done. If another thread sees the mutex is locked while it tries to access it, it will wait. The waiting can be a spin loop (see the [spin crate](https://docs.rs/spin/0.5/spin/struct.Mutex.html)), but it is much better to ask the operating system for help: "notify me know when it gets unlocked". Because the fast path involves just an atomic write, this makes it a very efficient primitive.

Sadly not all implementations are modern, and some come with restrictions. For example Posix requires the mutex to be immovable even when not locked, forcing the standard library to put any mutex in an allocation on the heap any not allowing its use in statics. A custom implementation such as the one in [parking_lot](https://crates.io/crates/parking_lot) fixes these issues.

A `RefCell` panics when you try to take two mutable borrows in the same thread. A `Mutex` useally does not detect this case, and will just start waiting to acquire the lock. But because it is the current thread that holds the lock, it will never wake up again (and block any thread that tries to lock the `Mutex` after that).

### RW lock
Simplified API ([see documentation](https://doc.rust-lang.org/std/sync/struct.RwLock.html), the actual API is a bit more complex because it allows the `RwLock` to be poisoned):
```rust
impl<T: ?Sized> RwLock<T> {
    pub fn read(&self) -> RwLockReadGuard<T>
    pub fn write(&self) -> RwLockWriteGuard<T>
}
unsafe impl<T: ?Sized + Send + Sync> Sync for RwLock<T> {}

impl<T: ?Sized> Deref for RwLockReadGuard<T> { /* ... */ }
impl<T: ?Sized> Drop for RwLockReadGuard<'_, T> { /* ... */ }

impl<T: ?Sized> DerefMut for RwLockWriteGuard<T> { /* ... */ }
impl<T: ?Sized> Deref for RwLockWriteGuard<T> { /* ... */ }
impl<T: ?Sized> Drop for RwLockWriteGuard<'_, T> { /* ... */ }

/* optional */
impl<T: ?Sized> RwLockWriteGuard<T> {
    pub fn downgrade(self) -> RwLockReadGuard<T>
}
```

A _reader-writer lock_ is the full-featured multithreaded alternative to `RefCell`. An `RwLock` will allow any number of readers to acquire the lock as long as a writer is not holding the lock.

There can be several strategies to implement an RW lock. Imagine this case: multiple threads keep taking `read` locks, and one thread wants to take a `write` lock. Should the reads be able to block the write indefinitely? [Wikipedia](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) lists three priority policies:
> - _Read-preferring RW locks_ allow for maximum concurrency, but can lead to write-starvation if contention is high.
> - _Write-preferring RW locks_ avoid the problem of writer starvation by preventing any new readers from acquiring the lock if there is a writer queued and waiting for the lock … The downside is that write-preferring locks allows for less concurrency in the presence of writer threads.
> - _Unspecified priority RW locks_ does not provide any guarantees with regards read vs. write access.

The standard library explicitly does not specify the priority policy, it is just a thin wrapper around what the operating system provides. `parking_lot` [documents](https://docs.rs/parking_lot/0.10/parking_lot/type.RwLock.html) it offers a _write-preferring RW lock_. The [spinlock-based implementation](https://mvdnes.github.io/rust-docs/spin-rs/spin/struct.RwLock.html) in `spin` is an example of a _read-preferring RW lock_.

A nice extra that `parking_lot` offers is a [`downgrade`](https://docs.rs/lock_api/0.3.2/lock_api/struct.RwLockWriteGuard.html#method.downgrade) method on its `RwLockWriteGuard` to turn it into a `RwLockReadGuard`, without releasing the lock in the meantime.

### Reentrant Mutex
Basic API ([documentation](https://docs.rs/lock_api/0.3/lock_api/struct.ReentrantMutex.html)):
```rust
impl<T: ?Sized> ReentrantMutex<T> {
    pub fn lock(&self) -> ReentrantMutexGuard<T>
}
unsafe impl<T: ?Sized + Send> Sync for ReentrantMutex<T> {}

impl<T: ?Sized> Deref for ReentrantMutexGuard<T> { /* ... */ }
impl<T: ?Sized> Drop for ReentrantMutexGuard<'_, T> { /* ... */ }
```

A reentrant mutex may be locked more than once by the same thread, typically because of recursion. The smart pointer returned when locking the `ReentrantMutex` can't be mutable, because it may not be unique. So it does not provide interior mutability by itself. But it does give the guarantees that allow using it with a single-threaded one, like `RefCell`, because:
- it can hand out multiple shared references,
- its wrapped type does not have to be `sync`, because the references are all handed out within the same thread.

Reentrant mutexes are considered a code smell, as they typically indicate you are holding a lock too long. But there are valid uses, like [`Stdout`](https://github.com/rust-lang/rust/blob/1.40.0/src/libstd/io/stdio.rs#L400) which may be locked by a thread, while `println!` will also lock it for each write.

A reentrant mutex requires a little more bookkeeping than a normal mutex. It has to keep track of the id of the thread that currently holds a lock, and it has to keep track of the number of locks held. The standard library does not expose its implementation of `ReentrantMutex` ([`remutex`](https://crates.io/crates/remutex) does for an old version), but `parking_lot` provides one.


## Mutability only during initialization

There is a common pattern where you need to do something at runtime to initialize a variable, but can't do so before sharing it with other threads. Statics are the prime example. [`lazy_static`](https://crates.io/crates/lazy_static) used to be the go-to solution, but multiple crates created and reinvented the `OnceCell` + `Lazy` solution, of which the [`once_cell`](https://crates.io/crates/once_cell) crate seems to have arrived at the cleanest API.

### OnceCell (unsync)
Basic API ([documentation](https://docs.rs/once_cell/1/once_cell/unsync/struct.OnceCell.html)):
```rust
impl<T> OnceCell<T> {
    pub fn get(&self) -> Option<&T>
    pub fn set(&self, value: T) -> Result<(), T>
    pub fn get_or_init<F: FnOnce() -> T>(&self, f: F) -> &T
}

impl<T, F> Lazy<T, F> {
    pub const fn new(init: F) -> Lazy<T, F>
}
impl<T, F: FnOnce() -> T> Deref for Lazy<T, F>
```

For the singlethreaded case a `OnceCell` can be in two states (which by the way allows a size optimization using niche filling like `Option`):
- uninitialized, which allow mutating it once with `set`;
- initialized, which allow taking multiple shared references with `get`.

Any time `get` is called while uninitialized it will return `None`, and any time `set` is called while initialized it will return `Err` (with the provided value). `get_or_init` is the nice combination: it can always return a reference, because it will initialize the `OnceCell` if necessary by running the provided closure.

The key insight that makes `OnceCell` safe is that a mutation may really be done only _once_. Recursive initialization (possibly accidental) within the closure of `get_or_init` therefore [must cause a panic](https://github.com/matklad/rfcs/blob/std-lazy/text/0000-standard-lazy-types.md#reference-level-explanation).

`Lazy` acts like a smart pointer build on top of `OnceCell`. The initialization closure is provided once with `new`, and run on the first use of `Lazy` through `Deref`.

[`mutate_once`](https://crates.io/crates/mutate_once) provides an variation based on the same principles that can hand out a mutable smart pointer first. But as soon as one normal reference is returned, it can never be mutated again.

### OnceCell (sync)
Basic API ([documentation](https://docs.rs/once_cell/1/once_cell/sync/struct.OnceCell.html)):
```rust
impl<T> OnceCell<T> {
    pub fn get(&self) -> Option<&T>
    pub fn set(&self, value: T) -> Result<(), T>
    pub fn get_or_init<F: FnOnce() -> T>(&self, f: F) -> &T
}
unsafe impl<T: Send + Sync> Sync for OnceCell<T> {}

impl<T, F> Lazy<T, F> {
    pub const fn new(init: F) -> Lazy<T, F>
}
impl<T, F: FnOnce() -> T> Deref for Lazy<T, F>
unsafe impl<T, F: Send> Sync for Lazy<T, F> where OnceCell<T>: Sync {}
```

The multi-threaded variant of `OnceCell` is almost the same. The state has to be kept in an atomic, and it has a third state: other threads may observe one thread is _running_ the initialization clusure. At that point they should wait, either using a spinloop but preferably with assistance from the OS, just like `Mutex`.

## Using a seperate 'owner' or 'token' to track references

A somewhat experimental concept is to bind a `Cell` to an 'owner' or 'token' type. If you have a shared reference to the owner type, you can create a shared reference to the inside of the bound `Cell`. And if you have a mutable reference to the owner type, you can also get one to the `Cell`.

The owner can be shared freely between threads and passed through channels. I suppose that if you already need to do some communication through channels, this is a nice solution to avoid also taking a mutex (or passing large amounts of data through the channel). _Still, when and how to use this pattern is not really clear to me, this description may not do it right._

The [`qcell` crate](https://crates.io/crates/qcell) explores this technique (see also the [initial post on Reddit](https://reddit.com/r/rust/comments/auqedt/compilerchecked_alternative_to_refcell/) and the [announcement](https://users.rust-lang.org/t/new-crate-qcell-a-statically-checked-alternative-to-refcell/25756) on The Rust Programming Language Forum).

A problem is how to conjure up a unique owner. One option is to just use a unique integer (`QCell`). Another is to use your own zero-sized marker type (`TCell`). And a third is to use the unique lifetime of a closure (`LCell`).

### QCell
Basic API of `QCell`:
```rust
impl QCellOwner {
    pub fn new() -> Self
    pub fn cell<T>(&self, value: T) -> QCell<T>
    pub fn ro<'a, T>(&'a self, qc: &'a QCell<T>) -> &'a T
    pub fn rw<'a, T>(&'a mut self, qc: &'a QCell<T>) -> &'a mut T
    pub fn rw2<'a, T, U>(
        &'a mut self,
        qc1: &'a QCell<T>,
        qc2: &'a QCell<U>
    ) -> (&'a mut T, &'a mut U)
    pub fn rw3<'a, T, U, V>(
        &'a mut self,
        qc1: &'a QCell<T>,
        qc2: &'a QCell<U>,
        qc3: &'a QCell<V>
    ) -> (&'a mut T, &'a mut U, &'a mut V)
}

impl<T> QCell<T> {
    pub const fn new(owner: &QCellOwner, value: T) -> QCell<T>
}
impl<T: Send + Sync> Sync for QCell<T>
```

A new `QCell` needs a reference to a `QCellOwner` when it is created by either `QCellOwner::cell` or `QCell::new`. One ugly part of the API is that the methods to take a reference to the `QCell`, `ro`, and to take a mutable reference, `rw`, live on the `QCellOwner`.

While one cell is borrowed mutably, it is as if the owner is mutably borrowed. No other cells bound to the same `QCellOwner` can be borrowed at the same time. That is why if you need multiple references with at least one mutable, you need to take them all at once with `rw2` or `rw3`.

### TCell and LCell

The main difference between `QCell`, `TCell` and `LCell` is how the owner is constructed. For `QCell` it is as simple as `QCellOwner::New()`.

For `TCell` you need to provide a marker type, that must be unique within the process, which is double-checked at runtime (there also is `TLCell`, which must be unique per thread):
```rust
struct Marker1;
let owner1 = TCellOwner::<Marker1>::new();
```

An `LCellOwner` can't be simply constructed. You have to run the code that uses it in a closure, where the owner is an argument to the closure. [`GhostCell`](https://github.com/ppedrot/kravanenn/blob/master/src/util/ghost_cell.rs) initially explored this approach. The lifetime of the closure is the id of the owner:
```rust
LCellOwner::scope(|mut owner| {
    /* use `owner` */
});
```

`QCell` is the most flexible. `TCell` and `LCell` move more checks to compile time, but require more annotations or a certain code structure:

Cell                | Owner ID    | Cell overhead | Owner check  | Owner creation check
--------------------|-------------|---------------|--------------|---------------------
`QCell`             | integer     | `u32`         | Runtime      | Runtime
`TCell` or `TLCell` | marker type | none          | Compile-time | Runtime
`LCell`             | lifetime    | none          | Compile-time | Compile-time

## Overview

You made it to the end ;-). How do the various API's compare?

`Cell`-like, allowing no references inside:

| type             | requirements          | `Sync`? | restrictions
|------------------|-----------------------|---------|--------------------------------------------
| `Cell<T>`        | `Copy`                | no      | value must be copied or moved
| `Cell<T>`        | (none)                | no      | value must be moved with `replace` or `take`
| `Atomic<T>`*     | `Send`, integer-sized | yes     | like `Cell`
| `AtomicCell<T>`* | `Send`                | yes     | like `Cell`

_* Note: I used `Atomic<T>` to describe atomic operations, and `AtomicCell<T>` to describe a lock-based fallback._

`RefCell`-like, ensuring at runtime there is only one `&mut` reference:

| type                         | requirements    | `Sync`? | `&` | `&mut` | restrictions
|------------------------------|-----------------|---------|-----|--------|----------------------
| `RefCell<T>`                 | (none)          | no      | yes | yes    | may panic
| `Mutex<T>`                   | `Send`          | yes     | no  | yes    | may block or deadlock
| `RwLock<T>`                  | `Send` + `Sync` | yes     | yes | yes    | may block or deadlock
| `ReentrantMutex<RefCell<T>>` | `Send`          | yes     | yes | yes    | may block or panic


`OnceCell`, only mutable during initialization:

| type                  | requirements    | `Sync`? | `&` | restrictions
|-----------------------|-----------------|---------|-----|-----------------------------
| `unsync::OnceCell<T>` | (none)          | no      | yes | only mutable once
| `sync::OnceCell<T>`   | `Send` + `Sync` | yes     | yes | may block, only mutable once

This post concentrated on the possible API choices, and only touched upon the choices that can be made for the synchronization primitives.

Do you know of any other patterns? Or found a mistake? Please let me know.
