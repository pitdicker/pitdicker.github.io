---
layout: post
title: Creative methods on interior mutability types
---

In the previous post [Interior mutability patterns](../Interior-mutability-patterns/) we looked at four different conventions that can be used to safely use interor mutability. We didn't look further than the basic API, that was already plenty of material to cover.

Yet there are some creative methods that push the limit of those abstractions. Let's explore them!

On container types:

| method              |  `Cell` | `Atomic`¹| `AtomicCell`¹| `RefCell`  | `Mutex` | `RwLock` | `OnceCell` | `QCell` | `TCell` | `LCell`
|---------------------|---------|----------|--------------|------------|---------|----------|------------|---------|---------|--------
| `into_inner`        | yes     | yes      | yes          | yes        | yes     | yes      | yes        | yes*    | yes*    | yes*
| `get_mut`           | yes     | —        | yes*         | yes        | yes     | yes      | yes        | yes*    | yes*    | yes*
| `as_ptr`            | yes     | yes      | yes          | yes        | yes     | yes      | yes        | yes     | yes     | yes
| `from_mut`          | yes     | —        | yes*         | (`BoCell`) | —       | —        | —          | —       | yes*    | yes*
| `replace`           | yes     | yes      | yes          | yes        | —       | —        | —          | yes*    | yes*    | yes*
| `swap`              | yes     | yes      | yes          | yes        | —       | —        | —          | —       | —       | —
| `update`            | yes     | —        | —            | yes        | —       | —        | —          | —*      | —*      | —*
| `as_slice_of_cells` | yes     | —        | —            | —          | —       | —        | —          | —       | yes*    | yes*
| `with`              | —       | —        | —            | —          | maybe   | maybe    | —          | —       | —       | —

On smart pointers:

| method          | `Ref`  | `RefMut` | `MutexGuard` | `RwLockReadGuard` | `RwLockWriteGuard` | `ReentrantMutexGuard`
|-----------------|--------|----------|--------------|-------------------|--------------------|----------------------
| `bump`          | —      | —        | yes*         | yes*              | yes*               | yes
| `unlocked`      | —      | —        | yes*         | yes*              | yes*               | yes
| `Condvar::wait` | —      | —        | yes          | yes*              | yes*               | yes*
| `map`           | yes    | yes      | yes*         | yes*              | yes*               | yes
| `map_split`     | yes    | yes      | yes*         | yes*              | yes*               | yes*
| `try_map`       | yes*   | yes*     | yes*         | yes*              | yes*               | yes
| `downgrade`     | —      | —        | —            | —                 | yes*               | —
| `upgrade`       | yes*   | —        | —            | maybe             | —                  | —

_¹: In this post I use `Atomic<T>` to describe a wrapper based on atomic operations, and `AtomicCell<T>` to describe a lock-based solution for larger types._

_*: possible, but not implemented (or not implemented in the standard library or main crate)_

<!--more-->

We could place these methods in 9 categories:
- [Bypass conventions if you have exclusive access](#bypass-conventions-if-you-have-exclusive-access)
- [Temporary wrap a value to provide interior mutability](#temporary-wrap-a-value-to-provide-interior-mutability)
- [`mem::replace`, but for interior mutability](#memreplace-but-for-interior-mutability)
- [Updating a value (single-threaded wrappers only)](#updating-a-value-single-threaded-wrappers-only)
- [References in `Cell` if the structure is 'flat'](#references-in-cell-if-the-structure-is-flat)
- [Temporary lending out a mutable reference to another thread](#temporary-lending-out-a-mutable-reference-to-another-thread)
- [Refining smart pointers](#refining-smart-pointers)
- [Downgrading or upgrading a smart pointer](#downgrading-or-upgrading-a-smart-pointer)
- [Scopes instead of smart pointers](#scopes-instead-of-smart-pointers)

## Bypass conventions if you have exclusive access

For many uses of types with interior mutability there is a point in the program where the value is not yet shared, and you either own the value, or have exclusive access. Or there is a point where you are done sharing, for example in the `Drop` implementation of a type.

If you have exclusive, mutable access, you don't need interior mutability. In that case you can just use the wrapped value, without having to uphold the conventions the wrapper normally asks for in order to use the value with multiple references.

Basic API:
```rust
impl<T: ?Sized> Cell<T> {
    fn into_inner(self) -> T
    fn get_mut(&mut self) -> &mut T
    const fn as_ptr(&self) -> *mut T
}
```

`into_inner` just returns the wrapped value by consuming the wrapper type.

### `get_mut`
If you can provide a mutable reference to the wrapper type, `get_mut` gives a mutable reference to the wrapped value. This is just as possible with synchronization primitives like `Mutex` and `RwLock`, which don't need to do any locking in that case.

A notable exception for `get_mut` is a wrapper that treats [small types as atomic integers](../Interior-mutability-patterns#atomics), here named `Atomic<T>` (although there is no crate that provides such an abstraction without fallback for larger types).

`Atomic<T>` is currently unsound when used on types with padding bytes. This could be solved in all but one case if there is some kind of "atomic freeze operation", which "freezes" the value of the padding bytes before any operation on them. The 'but one' case is `get_mut`: it would allow user code to change the value without freezing the padding bytes.

For now this is a bit of a theoretical concern, as there is no such thing as a freeze operation. The discussion around a freeze operation has died down after a generic [`ptr::freeze`](https://github.com/rust-lang/rust/pull/58363) method for uninitialized memory turned out to be impossible, although that doesn't make it impossible for just padding bytes of small structs.

### `as_ptr`
`as_ptr` is a method that is always safe to call, which will return a raw pointer to the wrapped value. Dereferencing the pointer however requires an unsafe block. Any use of the pointer has to uphold any conventions the wrapper type upholds, which is nigh impossible because you don't have access to the internal state of the wrapper. That makes `as_ptr` basically only usable on `Cell`, or in situations where you already have exclusive access, like `into_inner` and `get_mut`.

An `as_ptr` method was not available for normal integer atomics. Such a function may be useful for FFI, where the receiving library also promises to do only atomic operations on the pointed value. I made a PR, and it is now available on nightly as [`as_mut_ptr`](https://doc.rust-lang.org/nightly/std/sync/atomic/struct.AtomicUsize.html#method.as_mut_ptr).

## Temporary wrap a value to provide interior mutability

Basic API:
```rust
impl<T: ?Sized> Cell<T> {
    fn from_mut(t: &mut T) -> &Cell<T>
}
```

If an interior mutability wrapper does not need any extra fields, and does not place any extra restrictions such as a certain alignment, it can be made `#[repr(C)]` or [`#[repr(transparent)]`](https://doc.rust-lang.org/1.26.2/unstable-book/language-features/repr-transparent.html).

Such as wrapper can be put around any value where you have mutable access to, allowing you to temporary create multiple references in combination with interior mutability. _This is a useful pattern if you have hard to solve problems with the borrow checker that require multiple mutable references._

[RFC 1789](https://rust-lang.github.io/rfcs/1789-as-cell.html) established this pattern with [`Cell::from_mut`](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.from_mut). 
It can also be supported by `TCell` and `LCell` of the [`qcell` crate](https://crates.io/crates/qcell). A wrapper like `AtomicCell` could support it if it always only used a locking method (i.e. no optimization with atomic operations for small types), and uses a set op global locks (instead of putting a lock together with the value).

For `Atomic<T>` (for small integer-sized types) `from_mut` is not possible, because the alignment may not be correct. Besides requiring state it wouldn't make any sense for `OnceCell`  to have `from_mut`: the only time when it provides mutability is when the value is not yet initialized, and you can't have mutable references to uninitialized memory.

`RefCell`, `Mutex` etc. don't support wrapping an arbitrary type because they need to hold some extra state. The recent [`BoCell`](https://docs.rs/bo_cell/0.1/bo_cell/struct.BoCell.html) is a simple solution for `RefCell`. Instead of storing the state with the wrapped value, you can just store the state somewhere else with a pointer to the value. The signature of `BoCell::new` matches that of `from_mut`.

One small [missed opportunity](https://github.com/rust-lang/rust/issues/43038#issuecomment-521013438) for `Cell::from_mut` was to have it return `&mut Cell<T>`, which would have allowed using the returned value again with `Cell::get_mut`.

## `mem::replace`, but for interior mutability

Basic API:
```rust
impl<T: ?Sized> Cell<T> {
    fn replace(&self, val: T) -> T
    fn swap(&self, other: &Cell<T>)
}
```

For `Cell` we have [already seen](../Interior-mutability-patterns#atomics) the `replace` method, which allows reading the old value and setting an new value in one operation. The same is possible for `Atomic<T>` and `AtomicCell<T>`.

`RefCell` offers a `replace` method too, with the restriction that it panics when there are any active borrows. In theory `Mutex` and `RwLock` could also support it, but until now the maintainers wisely took the decision to not add any methods that would secretly take a lock.

What if you want to swap the value of two Cells? You could do so by using `replace` twice in combination with a temporary value. `Cell` and `RefCell` both offer a `swap` method for convenience, and for large types, performance. I suppose it could also be offered by the types of the `qcell` crate, but that may get a too complex because you would not only pass two cells but also have to handle two owners.

## Updating a value (single-threaded wrappers only)

Basic API:
```rust
impl<T: Copy> Cell<T> {
    fn update<F>(&self, f: F) -> T
        where F: FnOnce(T) -> T
}

impl<T: ?Sized> RefCell<T> {
    fn replace_with<F>(&self, f: F) -> T
        where F: FnOnce(&mut T) -> T
}
```

One trick that is available for single-threaded types is [`Cell::update`](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.update) ([unstable](https://github.com/rust-lang/rust/issues/50186)) or [`RefCell::replace_with`](https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.replace_with). These methods run a closure with the current value in the cell as input, and write the result to the value when done.

A problem is: what happens when you try to access the cell while inside the closure? `Cell::update` currently has a `Copy` bound. It will pass a copy of the value to the update closure. Reads from the cell during the closure will return the old value, and writes will be overwritten after the closure is done.

An alternative is requiring a `Default` bound, so that `Cell::update` `takes` out the value, reads from the cell during the closure will see a default value, and a new value will be put back in after the closure is done. Because both options are equally valid and both just for convenience, I wonder how long it will take to reach a decision and stabilize the function.

For `RefCell::replace_with` it is easy: borrowing it again inside the closure causes a panic, just like other invalid borrows. I wonder if there is much use for `replace_with`, but it exists.

For synchronization primitives something like `Cell::update` is tricky, other threads could have modified the value while the closure is running. You really want to use `compare_and_swap`-like methods instead, or something like the experimental [`fetch_update`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html#method.fetch_update) on atomic integers.

For synchronization primitives something like `RefCell::replace_with` is also not recommended, because that would mean secretly taking a lock.

For `OnceCell` it doesn't make sense to update the value, because the only time its value can be mutated is while it is uninitialized. An alternative could be [`OnceFlipCell`](https://github.com/matklad/rfcs/blob/std-lazy/text/0000-standard-lazy-types.md#why-oncecell-as-a-primitive), which comes with an initial state that can be flipped to a usable state. But I really don't see much use for such a type.

## References in `Cell` if the structure is 'flat'

The basic convention that makes `Cell` safe is to never hand out references to its wrapped value. It turns out here is a case where taking a reference to inside the cell doesn't violate its soundness. That would be really useful for larger types, as it allows you to read or write only part of the value.

Let's call the structure of a value 'flat' (see [this comment](https://github.com/rust-lang/rfcs/pull/1789#issuecomment-260209527)) if the references don't go under indirections (`Box` or other wrapper types) or branches (`enum`s). Then a write to `Cell` can't change the structure, there is only one possible structure. A write to the entire `cell` will mutate the contents, but it can never switch a part to which we hold a reference to another enum variant, deallocate the referenced memory, or otherwise make the inner reference invalid.

We do have to make sure the inner references also follow the same conventions for interior mutability, meaning we have also have to wrap them in `Cell`s. Just like [temporary wrapping a value to provide interior mutability](#temporary-wrap-a-value-to-provide-interior-mutability).

That brings us to a few safe conversions:
- tuples: `&Cell<(A, B)>` → `&(Cell<A>, Cell<B>)`
- arrays: `&Cell<[A; N]>` → `&[Cell<A>; N]`
- slices: `&Cell<[A]>` → `&[Cell<A>]`
- structs: `&Cell<struct>` → `&Cell<field>` (if the field is not private)

Only the conversion of slices is currently implemented, with [`Cell::as_slice_of_cells`](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.as_slice_of_cells):
```rust
impl<T> Cell<[T]> {
    fn as_slice_of_cells(&self) -> &[Cell<T>]
}
```

Are there any other internal mutability types that can follow this trick? The type has to support the same operation as `from_mut`, and it should not hand out a regular mutable reference from the parent cell and from the inner cell at the same time (`Cell` never hands out any, so that is easy).

`AtomicCell` also doesn't hand out references, but uses the address of the wrapped value to synchronize accesses with other threads. Even if we take a reference to something inside it and wrap it in an `AtomicCell` again, the address will not be the same. So I don't think it is something that can be supported.

`TCell` and `LCell` could implement `from_mut`, and can also ensure there a mutable reference is unique. The owner ensures either all references are shared, or that there is only one mutable reference. Multiple mutable references can be created with [`LCellOwner::{rw2, rw3}`](https://docs.rs/qcell/0.4.0/qcell/struct.LCellOwner.html#method.rw2), but those methods check at runtime the adresses are different, and can be extended to check the addresses don't fall within the size of one of the values.

## Temporary lending out a mutable reference to another thread
Simplified API:
```rust
impl<'a, T: ?Sized> MutexGuard<'a, T> {
    fn bump(s: &mut Self)
    fn unlocked<F, U>(s: &mut Self, f: F) -> U
        where F: FnOnce() -> U
}

impl Condvar {
    fn wait<T: ?Sized>(&self, mutex_guard: &mut MutexGuard<T>)
}
```

### `bump` and `unlocked`

With a `Mutex` you may want to temporary release your lock on the primitive to let another thread do some work.

`parking_lot` offers a [`MutexGuard::bump`](https://docs.rs/lock_api/0.3/lock_api/struct.MutexGuard.html#method.bump) method that lets you temporary give up your lock on the mutex, and block until the lock can be acquired again. The similar [`MutexGuard::unlocked`](https://docs.rs/lock_api/0.3/lock_api/struct.MutexGuard.html#method.unlocked) method lets you run a closure in the meantime (and block afterwards). This is functionally equivalent to lacking and unlocking the mutex, but it can be much more efficient in the case where there are no waiting threads.

`bump` and `unlocked` take exclusive access to the smart pointer. Another thread which acquires the mutex has mutable access, so in a way you can view those two methods as lending out a mutable reference to another thread.

Is `bump` also sound for `RwLock` and `ReentrantMutex`? `RwLockWriteGuard` has exclusive mutable access, so it can lend out that access to another thread with `bump`. But one thread can hold multiple instances of `RwLockReadGuard` or `ReentrantMutexGuard`, in which case `bump` can't guarantee exclusive access.

Still they are also safe, because now it is not the type system that guarantees exclusive access, but the synchronization primitive. If the current thread holds more than one lock, only one will be released. Then another thread won't be able to get a lock for write access (or in the case of `ReentrantMutex`, won't be able to get a lock at all). `bump` will basically have no effect if there are multiple instances of `RwLockReadGuard` or `ReentrantMutexGuard`.

### Condition variables

Waiting on a condition variable with a mutex with [`Condvar::wait`](https://doc.rust-lang.org/std/sync/struct.Condvar.html#method.wait) also gives up the lock, but the thread doesn't wake up again until notified.

You `wait` on the `Condvar`, giving it a mutable reference to the `MutexGuard`, which temporary releases the lock. Another thread acquires the lock, does the work you are waiting on, calls [`Condvar::notify_*`](https://doc.rust-lang.org/std/sync/struct.Condvar.html#method.notify_one) and releases the lock to wake you up. You now again hold the lock.

Temporary lending out mutable access to another thread by waiting on a `Condvar` with `MutexGuard` is safe, for the same reasons as `bump` and `unlocked`. The standard library and `parking_lot` only support this combination, using a `Condvar` in with a `MutexGuard`.

The [`shared-mutex` crate](https://crates.io/crates/shared-mutex) has an alternative RW lock implementation that is designed to be useable with condition variables. Again waiting with a `SharedMutexWriteGuard` is sound, as we can lend a unique reference. It is also correct if the thread holds only one `SharedMutexReadGuard`. And if the thread holds multiple `SharedMutexReadGuard`, only one will be unlocked and the thread will wait until notified. As another thread can't acquire a write lock now, we will probably have a deadlock but no unsoundness.

There is currently no crate (that I know of) that supports waiting on a `Condvar` in combination with a `ReentrantMutex`. It would probably quickly lead to deadlocks, because if it was possible to avoid the chance of holding multiple locks, you should not be using `ReentrantMutex`.

In the common `ReentrantMutex<RefCell>` case there is a decoupling between holding a number of locks, and holding references. If those two were more closely integrated it would be possible to temporary give up all locks a thread holds if it only has one reference active, when waiting on a `Condvar`. And to panic otherwise.

## Refining smart pointers

Basic API (see documentation of [`Ref`](https://doc.rust-lang.org/std/cell/struct.Ref.html), [`RefMut`](https://doc.rust-lang.org/std/cell/struct.RefMut.html)):
```rust
impl<T: ?Sized> RefCell<T> {
    fn borrow(&self) -> Ref<T>
    fn borrow_mut(&self) -> RefMut<T>
}

impl<'b, T: ?Sized> Ref<'b, T>{
    fn map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
        where U: ?Sized,
              F: FnOnce(&T) -> &U
    fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>)
        where U: ?Sized,
              V: ?Sized,
              F: FnOnce(&T) -> (&U, &V)
    fn try_map<U, F>(orig: Ref<'b, T>, f: F) -> Result<Ref<'b, U>, Self>
        where U: ?Sized,
              F: FnOnce(&T) -> Option<&U>
    fn clone(orig: &Ref<'b, T>) -> Ref<'b, T>
}

impl<'b, T: ?Sized> RefMut<'b, T> {
    fn map<U, F>(orig: RefMut<'b, T>, f: F) -> RefMut<'b, U>
        where U: ?Sized,
              F: FnOnce(&mut T) -> &mut U
    fn map_split<U, V, F>(orig: RefMut<'b, T>, f: F) -> (RefMut<'b, U>, RefMut<'b, V>)
        where U: ?Sized,
              V: ?Sized,
              F: FnOnce(&mut T) -> (&mut U, &mut V)
    fn try_map<U, F>(orig: RefMut<'b, T>, f: F) -> Result<RefMut<'b, U>, Self>
        where U: ?Sized,
              F: FnOnce(&mut T) -> Option<&mut U>
}
```

Note that all these methods on smart pointers don't take `self` as argument, but are associated methods instead. A normal method would interfere with methods of the same name on the contents of a `RefCell` used through `Deref`.

### `Ref::map`
If you have a smart pointer type to inside a `RefCell`, you can take normal references to parts of the wrapped type. But `Ref::map` allows you to refine the smart pointer to point to a part of the wrapped type.

`map` was also once implemented for `MutexGuard`, `RwLockReadGuard` and `RwLockWriteGuard`, but [removed](https://github.com/rust-lang/rust/issues/27746#issuecomment-180458770). The problem is the interaction with `Condvar`. Notice that if we would refine the smart pointer with `MutexGuard::map` to only part of the wrapped value, through `Condvar` we would give the other thread also mutable access to _only part of the value_. But the other thread gets access to the _entire_ wrapped value. That can easily lead to things like reading from deallocated memory, as discovered in [this comment](https://github.com/rust-lang/rust/pull/30834#issuecomment-180284290).

The solution is to make `map` return a different type so it can't be used as argument for `Condvar::wait`. `parking_lot` implements this solution, its [`MutexGuard::map`](https://docs.rs/lock_api/0.3/lock_api/struct.MutexGuard.html#method.map) returns a `MappedMutexGuard`. Similarly the `map` methods on [`RwLockWriteGuard`](https://docs.rs/lock_api/0.3/lock_api/struct.RwLockWriteGuard.html), [`RwLockReadGuard`](https://docs.rs/lock_api/0.3/lock_api/struct.RwLockReadGuard.html) and [`ReentrantMutexGuard`](https://docs.rs/lock_api/0.3/lock_api/struct.ReentrantMutexGuard.html) should return another type, because they can also [temporary lend out a mutable reference](#temporary-lending-out-a-mutable-reference-to-another-thread) to the entire value to another thread with `bump`.

### `RefMut::map_split`
While `Ref::map` is nice, `RefMut::map_split` is where it gets really interesting in my opinion. It allows you to refine a smart pointer to _two_ parts of the wrapped value. This is the only way to get more than one mutable reference to inside the `RefCell`.

Just like `map`, `map_split` is not implemented for `Mutex` and `RwLock` in the standard library. `parking_lot` also doesn't support `map_split` yet. And because the concept of multiple mutable references goes pretty deep, that may take a long time, if ever, to be supported.

### `Ref::try_map`
Sometimes you want to refine the smart pointer to part of the value, but don't know yet if it exists. Examples are an enum, vector or hashmap. If it is undesirable to do the lookup twice, once to confirm it exists, and once in `map`, something like `try_map` comes in handy.

It was at some time available inside the standard library as the unstable method [`Ref::filter_map`](https://doc.rust-lang.org/1.7.0/std/cell/struct.Ref.html#method.filter_map). The [`ref_filter_map` crate](https://crates.io/crates/ref_filter_map) provides the functionality for `RefCell` after it got removed from the standard library by smuggling around a raw pointer.

`parking_lot` has a [`try_map`](https://docs.rs/lock_api/0.3/lock_api/struct.RwLockReadGuard.html#method.try_map) method for all its guards, just as `shared_mutex` has with [`result_map`](https://docs.rs/shared-mutex/0.3.1/shared_mutex/struct.MappedSharedMutexWriteGuard.html#method.result_map). In my opinion that API is superior to the original `filter_map`, because it returns a `Result`. That way the method can return either a refined reference, or the original one.

### Recover

Both `parking_lot` and `shared_mutex` return `Mapped*` types from their map methods. `shared_mutex` has a potentially useful method: it can [`recover`](https://docs.rs/shared-mutex/0.3/shared_mutex/struct.MappedSharedMutexReadGuard.html#method.recover) a reference to the entire value from a mapped reference, without releasing the lock, if you can also provide a reference to the mutex / RW lock.

It is worth noting that a library can't provide both `split_mut` and `recover` on a mutable (mapped) smart pointer. `split_mut` would allow multiple mutable references to exist, pointing to different parts of the value. Recovering from one a mutable reference to the entire value would alias the other ones.

## Downgrading or upgrading a smart pointer
Basic API:
```rust
impl<'b, T: ?Sized> RefMut<'b, T> {
    fn downgrade(orig: RefMut<'b, T>) -> Ref<'b, T>
}

impl<'b, T: ?Sized> Ref<'b, T> {
    fn upgrade(orig: Ref<'b, T>) -> RefMut<'b, T>
}
```

### Downgrade

`RwLockWriteGuard` in `parking_lot` has a [`downgrade`](https://docs.rs/lock_api/0.3/lock_api/struct.RwLockWriteGuard.html#method.downgrade) method which turns it into a `RwLockReadGuard`. It is instructive to explore why `RefCell`s `RefMut` can't provide a similar method to turn it into a `Ref`.

The signature of the closure used in `RefMut::map` is `FnOnce(&mut T) -> &mut U`. This gives it an interesting property: it allows you to [bypass conventions](#bypass-conventions-if-you-have-exclusive-access) of wrapped interior mutability types because you have exclusive access.

Because `RefMut::map` made the promise to wrapped types that its reference is unique, the returned `RefMut` has to remain unique. It can't be turned into a `Ref`, of which multiple can exist. Example of how it can go wrong:
```rust
let refcell = RefCell::new(Cell::new(10u32));
let ref_mut = refcell.borrow_mut();
RefMut::map(ref_mut, |x| x.get_mut());
let ref1 = RefMut::downgrade(ref_mut);
let cell_ref = &*ref1;
let ref2 = refcell.borrow();
// We can now mutate the `Cell` through `ref2` while there also exists a
// reference to its interior.
```

For `RwLockWriteGuard` however `downgrade` is sound, because it's `map` method returns another type. But that returned type, `MappedRwLockWriteGuard`, now has to remain unique, so [`MappedRwLockWriteGuard::downgrade`](https://docs.rs/lock_api/0.3/lock_api/struct.MappedRwLockWriteGuard.html#method.downgrade) is unsound.

### Upgrade

For `RefCell` a `Ref::upgrade` method can be simple. `Ref::map` can't traverse into wrapped interior mutability types because it doesn't guarantee unique access, so we don't have to problem `RefMut::downgrade` has. And if there are multiple shared references to the interior of the `RefCell` when upgrading, the convention is to panic. But there doesn't really seem to be an advantage to warrant an `upgrade` method.

For `RwLock` an `RwLockReadGuard::upgrade` method can be useful, as it allows starting an operation without blocking other threads from reading, and only upgrading to a writer lock when necessary. If other threads hold read locks, `upgrade` will have to block until they are done (just as when acquiring a write lock). And if the current thread holds more than one reader lock, it will deadlock.

The problem is: what happens if two reader locks attempt to upgrade at the same time? As both hold on to their current lock and wait until there is only one active lock, they deadlock. And contrary to other deadlocks wich can be considered programmer errors, this one may be unavoidable.

Two possible solutions:
- `upgrade` must be fallible, where one reader releases its read lock when `upgrade` fails, so the other can succeed.
- Establish before the upgrade moment that a reader is upgradable, and allow only one upgradable reader.

`parking_lot` has an [`RwLockUpgradableReadGuard`](https://docs.rs/lock_api/0.3.2/lock_api/struct.RwLockUpgradableReadGuard.html), the second option. It can probably become the best solution, but it currently misses a number of nice conversions such as deriving it from a normal read lock, or working with mapped smart pointers.

### Combined with waiting on a condition variable

Basic API:
```rust
impl<'mutex, T: ?Sized> SharedMutexReadGuard<'mutex, T> {
    fn wait_for_write(self, cond: &Condvar)
        -> LockResult<SharedMutexWriteGuard<'mutex, T>>
    fn wait_for_read(self, cond: &Condvar) -> LockResult<Self>
}

impl<'mutex, T: ?Sized> SharedMutexWriteGuard<'mutex, T> {
    fn wait_for_write(self, cond: &Condvar) -> LockResult<Self>
    fn wait_for_read(self, cond: &Condvar)
        -> LockResult<SharedMutexReadGuard<'mutex, T>>
}
```

In the part on [condition variables](#condition-variables) we have seen waiting on those is an operation that ensures the smart pointer is unique. And when waiting the threads temporary give up their lock on the RW lock. That makes this a safe moment to convert a read lock to a write lock.

The [`SharedMutex`](https://docs.rs/shared-mutex/0.3/shared_mutex/struct.SharedMutex.html) RW lock implementation provides such combinations. `SharedMutexReadGuard::wait_for_write` will upgrade the lock after waiting, and `SharedMutexWriteGuard::wait_for_read` will downgrade it.

## Scopes instead of smart pointers

Smart pointers are not the only way to keep track of the number of references/locks handed out. So can scopes: methods that provide a closure with a reference, and do the bookkeeping before and after running the closure.

For `RefCell` it doesn't make any sense to use it with a closure to provide scoped access to the interior: its whole point is to be used in cases where the regular scope-based 'inherited mutability' doesn't work out. For `OnceCell`, `QCell`, `TCell` and `LCell` it also doesn't offer a real advantage, they can already hand out plain references.

Just like it is not sound to hand out smart pointers to a `Cell`, it is not sound to hand out references to its interior when inside a closure. A scope can keep track of how long the reference is alive, but nothing else. It doesn't prevent another reference to the `Cell` to be used inside the closure, resulting in a shared reference being able to observe a mutation.

But for synchronization primitives you typically want to keep the structure regarding mutability within a thread clear, dealing with concurrency is already hard enough. So here scope-based methods can be a good alternative, and scopes make it easier to control the lifetime of a lock than smart pointers.

Basic API, from [a PR](https://github.com/rust-lang/rust/pull/61976) attempting to add such methods to `Mutex` and `RwLock` in the standard library:

```rust
impl<T: ?Sized> Mutex<T> {
    fn with<U, F>(&self, f: F) -> U
        where F: FnOnce(&mut T) -> U
}

impl<T: ?Sized> RwLock<T> {
    fn with_read<U, F>(&self, func: F) -> U
        where F: FnOnce(&T) -> U
    fn with_write<U, F>(&self, func: F) -> U
        where F: FnOnce(&mut T) -> U
}
```

One issue that conplicated things is that `Mutex` and `RwLock` have a feature: if one thread holding a lock panics, it will poison the mutex / RW lock, signalling the contents are probably in an inconsistent state. To force you to check for this scenario, every time you acquire a lock it doesn't just return a `MutexGuard<T>`, but `LockResult<MutexGuard<T>>` (which is commonly just unwrapped).

Because it is somewhat up for discussion how much value the ability to handle poisoned locks really has, compared to just panicking, and how to translate that into an API for working with scopes, the [PR was closed](https://github.com/rust-lang/rust/pull/61976#issuecomment-518337204).

For `Mutex` and `RwLock` in `parking_lot`, which don't support poisoning, I think a method like `with` can be a nice addition.

## Any other creativity?

Appearently rustaceans really like to push the boundaries, to explore the limits of what keeps these interior mutability conventions sound.

I can't think of any other directions, but I am sure this list will grow outdated at some point ;-).

Do you know of any other creative methods on interior mutability types?

### Revision history
- 2020-01-07: added 'Scopes instead of smart pointers'
- 2020-01-04: added `Ref::try_map` and `MappedRef::recover`
- 2020-01-04: added `Ref::upgrade`
- 2020-01-03: added 'Temporary lending out a mutable reference to another thread'
- 2020-01-02: added 'Downgrading a mutable smart pointer'
- 2020-01-01: `TCell` and `LCell` can also support `as_slice_of_cells`
