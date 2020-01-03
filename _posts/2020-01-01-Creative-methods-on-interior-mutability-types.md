---
layout: post
title: Creative methods on interior mutability types
---

In the previous post [Interior mutability patterns](../Interior-mutability-patterns/) we looked at four different conventions that can be used to safely use interor mutability. We didn't look further than the basic API, that was already plenty of material to cover.

Yet there are some creative methods that push the limit of those abstractions. Let's explore them!

| method              |  `Cell` | `Atomic`¹| `AtomicCell`¹| `RefCell`  | `Mutex` | `RwLock` | `OnceCell` | `QCell` | `TCell` | `LCell`
|---------------------|---------|----------|--------------|------------|---------|----------|------------|---------|---------|--------
| `into_inner`        | yes     | yes      | yes          | yes        | yes     | yes      | yes        | yes*    | yes*    | yes*
| `get_mut`           | yes     | —        | yes*         | yes        | yes     | yes      | yes        | yes*    | yes*    | yes*
| `as_ptr`            | yes     | yes      | yes          | yes        | yes     | yes      | yes        | yes     | yes     | yes
| `from_mut`          | yes     | —        | yes*         | (`BoCell`) | —       | —        | —          | —       | yes*    | yes*
| `replace`           | yes     | yes      | yes          | yes        | —       | —        | —          | yes*    | yes*    | yes*
| `swap`              | yes     | yes      | yes          | yes        | —       | —        | —          | —       | —       | —
| `update`            | yes     | —        | —            | yes        | —       | —        | —          | —*      | —*      | —*
| `Ref::map`          | —       | —        | —            | yes        | yes*    | yes*     | —          | —       | —       | —
| `Ref::map_split`    | —       | —        | —            | yes        | yes*    | yes*     | —          | —       | —       | —
| `as_slice_of_cells` | yes     | —        | —            | —          | —       | —        | —          | —       | yes*    | yes*
| `RefMut::downgrade` | —       | —        | —            | —          | —       | yes*     | —          | —       | —       | —

_¹: In this post I use `Atomic<T>` to describe a wrapper based on atomic operations, and `AtomicCell<T>` to describe a lock-based solution for larger types._

_*: possible, but not implemented (yet)_

<!--more-->

We could place these methods in six categories:
- [Bypass conventions if you have exclusive access](#bypass-conventions-if-you-have-exclusive-access)
- [Temporary wrap a value to provide interior mutability](#temporary-wrap-a-value-to-provide-interior-mutability)
- [`mem::replace`, but for interior mutability](#memreplace-but-for-interior-mutability)
- [Updating a value (single-threaded wrappers only)](#updating-a-value-single-threaded-wrappers-only)
- [Temporary lending out from a smart pointer](#temporary-lending-out-from-a-smart-pointer)
- [Refining smart pointers](#refining-smart-pointers)
- [References in `Cell` if the structure is 'flat'](#references-in-cell-if-the-structure-is-flat)
- [Downgrading a mutable smart pointer](#downgrading-a-mutable-smart-pointer)

## Bypass conventions if you have exclusive access

For many uses of types with interior mutability there is a point in the program where the value is not yet shared, and you either own the value, or have exclusive access. Or there is a point where you are done sharing, for example in the `Drop` implementation of a type.

If you have exclusive, mutable access, you don't need interior mutability. In that case you can just use the wrapped value, without having to uphold the conventions the wrapper normally asks for in order to use the value with multiple references.

Basic API:
```rust
impl<T: ?Sized> Cell<T> {
    pub fn into_inner(self) -> T
    pub fn get_mut(&mut self) -> &mut T
    pub const fn as_ptr(&self) -> *mut T
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
    pub fn from_mut(t: &mut T) -> &Cell<T>
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
    pub fn replace(&self, val: T) -> T
    pub fn swap(&self, other: &Cell<T>)
}
```

For `Cell` we have [already seen](../Interior-mutability-patterns#atomics) the `replace` method, which allows reading the old value and setting an new value in one operation. The same is possible for `Atomic<T>` and `AtomicCell<T>`.

`RefCell` offers a `replace` method too, with the restriction that it panics when there are any active borrows. In theory `Mutex` and `RwLock` could also support it, but until now the maintainers wisely took the decision to not add any methods that would secretly take a lock.

What if you want to swap the value of two Cells? You could do so by using `replace` twice in combination with a temporary value. `Cell` and `RefCell` both offer a `swap` method for convenience, and for large types, performance. I suppose it could also be offered by the types of the `qcell` crate, but that may get a too complex because you would not only pass two cells but also have to handle two owners.

## Updating a value (single-threaded wrappers only)

Basic API:
```rust
impl<T: Copy> Cell<T> {
    pub fn update<F>(&self, f: F) -> T
        where F: FnOnce(T) -> T
}

impl<T: ?Sized> RefCell<T> {
    pub fn replace_with<F>(&self, f: F) -> T
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

## Temporary lending out from a smart pointer
Simplified API:
```rust
impl Condvar {
    pub fn wait<T: ?Sized>(&self, mutex_guard: &mut MutexGuard<T>)
    // std:
    pub fn wait<'a, T>(&self, guard: MutexGuard<'a, T>) -> MutexGuard<'a, T>
}

impl<'a, T: ?Sized> MutexGuard<'a, T> {
    pub fn bump(s: &mut Self)
}
```

With a `Mutex` you may want to temporary release your lock on the primitive to let another thread do some work.

Woking with `Condvar` is a typical example. You lock a mutex, and [`wait`](https://doc.rust-lang.org/std/sync/struct.Condvar.html#method.wait) on the `Condvar`, which temporary releases the lock. Another thread acquires the lock, does the work you are waiting on, calls [`Condvar::notify_*`](https://doc.rust-lang.org/std/sync/struct.Condvar.html#method.notify_one) and releases the lock to wake you up. You now again hold the lock.

[`bump`](https://docs.rs/lock_api/0.3/lock_api/struct.MutexGuard.html#method.bump) is similar. You temporary give up your lock on the mutex (which has to be a fair unlock, otherwise the current thread can re-lock the mutex before another thread has had the chance to acquire the lock), and wake up when the lock can be acquired again. Only this time there is no other thread that gives a wake up signal. `bump` is functionally equivalent to calling `unlock_fair` followed by `lock`, however it can be much more efficient in the case where there are no waiting threads.

Both `Condvar` and `MutexGuard::bump` take exclusive access to the smart pointer, either through a mutable reference, or in the case of the standard library, by taking ownership of the smart pointer. Another thread which acquires the mutex has mutable access, so in a way you can view those two methods as lending out a mutable reference to another thread.

Is `bump` also sound for `RwLock` and `ReentrantMutex`? `RwLockWriteGuard` has exclusive mutable access, so it can lend out that access to another thread with `bump` or `Condvar` (if that last one were implemented). But one thread can hold multiple instances of `RwLockReadGuard` or `ReentrantMutexGuard`, in which case `bump` can't guarantee exclusive access. Still they are also safe, because if the current thread holds more than one lock only one will be released. Then another thread won't be able to get a lock for write access (or in the case of `ReentrantMutex`, won't be able to get a lock at all).

The standard library only supports the `Condvar` with `Mutex` case. `parking_lot` also supports the `bump` methods on the smart pointers of all three synchronization primitives.

## Refining smart pointers

Basic API (see documentation of [`Ref`](https://doc.rust-lang.org/std/cell/struct.Ref.html), [`RefMut`](https://doc.rust-lang.org/std/cell/struct.RefMut.html)):
```rust
impl<T: ?Sized> RefCell<T> {
    pub fn borrow(&self) -> Ref<T>
    pub fn borrow_mut(&self) -> RefMut<T>
}

impl<'b, T: ?Sized> Ref<'b, T>{
    pub fn map<U: ?Sized, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
        where F: FnOnce(&T) -> &U
    pub fn map_split<U: ?Sized, V: ?Sized, F>(
        orig: Ref<'b, T>, f: F
    ) -> (Ref<'b, U>, Ref<'b, V>)
        where F: FnOnce(&T) -> (&U, &V)
    pub fn clone(orig: &Ref<'b, T>) -> Ref<'b, T>
}

impl<'b, T: ?Sized> RefMut<'b, T> {
    pub fn map<U: ?Sized, F>(orig: RefMut<'b, T>, f: F) -> RefMut<'b, U>
        where F: FnOnce(&mut T) -> &mut U
    pub fn map_split<U: ?Sized, V: ?Sized, F>(
        orig: RefMut<'b, T>, f: F
    ) -> (RefMut<'b, U>, RefMut<'b, V>)
        where F: FnOnce(&mut T) -> (&mut U, &mut V)
}
```

### `Ref::map`
If you have a smart pointer type to inside a `RefCell`, you can take normal references to parts of the wrapped type. But `Ref::map` allows you to refine the smart pointer to point to a part of the wrapped type.

`map` was also once implemented for `MutexGuard`, `RwLockReadGuard` and `RwLockWriteGuard`, but [removed](https://github.com/rust-lang/rust/issues/27746#issuecomment-180458770). The problem is the interaction with `Condvar`. Notice that if we would refine the smart pointer with `MutexGuard::map` to only part of the wrapped value, through `Condvar` we would give the other thread also mutable access to _only part of the value_. But the other thread gets access to the _entire_ wrapped value. That can easily lead to things like reading from deallocated memory, as discovered in [this comment](https://github.com/rust-lang/rust/pull/30834#issuecomment-180284290).

The solution is to make `map` return a different type so it can't be used as argument for `Condvar::wait`. `parking_lot` implements this solution, its [`MutexGuard::map`](https://docs.rs/lock_api/0.3/lock_api/struct.MutexGuard.html#method.map) returns a `MappedMutexGuard`. Similarly the `map` methods on [`RwLockWriteGuard`](https://docs.rs/lock_api/0.3/lock_api/struct.RwLockWriteGuard.html), [`RwLockReadGuard`](https://docs.rs/lock_api/0.3/lock_api/struct.RwLockReadGuard.html) and [`ReentrantMutexGuard`](https://docs.rs/lock_api/0.3/lock_api/struct.ReentrantMutexGuard.html) should return another type, because they can also [temporary lend out a mutable reference](#temporary-lending-out-from-a-smart-pointer) to the entire value to another thread with `bump`.


### `RefMut::map_split`
While `Ref::map` is nice, `RefMut::map_split` is where it gets really interesting in my opinion. It allows you to refine a smart pointer to _two_ parts of the wrapped value. This is the only way to get more than one mutable reference to inside the `RefCell`.

Just like `map`, `map_split` is not implemented for `Mutex` and `RwLock` in the standard library. `parking_lot` also doesn't support `map_split` yet. And because the concept of multiple mutable references goes pretty deep, that may take a long time, if ever, to be supported.

Note that all these methods on smart pointers don't take `self` as argument, but are associated methods instead. A normal method would interfere with methods of the same name on the contents of a `RefCell` used through `Deref`.

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
    pub fn as_slice_of_cells(&self) -> &[Cell<T>]
}
```

Are there any other internal mutability types that can follow this trick? The type has to support the same operation as `from_mut`, and it should not hand out a regular mutable reference from the parent cell and from the inner cell at the same time (`Cell` never hands out any, so that is easy).

`AtomicCell` also doesn't hand out references, but uses the address of the wrapped value to synchronize accesses with other threads. Even if we take a reference to something inside it and wrap it in an `AtomicCell` again, the address will not be the same. So I don't think it is something that can be supported.

`TCell` and `LCell` could implement `from_mut`, and can also ensure there a mutable reference is unique. The owner ensures either all references are shared, or that there is only one mutable reference. Multiple mutable references can be created with [`LCellOwner::{rw2, rw3}`](https://docs.rs/qcell/0.4.0/qcell/struct.LCellOwner.html#method.rw2), but those methods check at runtime the adresses are different, and can be extended to check the addresses don't fall within the size of one of the values.

## Downgrading a mutable smart pointer
Basic API:
```rust
impl<'b, T: ?Sized> RefMut<'b, T> {
    pub fn downgrade(orig: RefMut<'b, T>) -> Ref<'b, T>
}
```

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

## Any other creativity?

Appearently rustaceans really like to push the boundaries, to explore the limits of what keeps these interior mutability conventions sound.

I can't think of any other directions, but I am sure this list will grow outdated at some point ;-).

Do you know of any other creative methods on interior mutability types?

### Revision history
- 2020-01-03: added 'Temporary lending out from a smart pointer'
- 2020-01-02: added 'Downgrading a mutable smart pointer'
- 2020-01-01: `TCell` and `LCell` can also support `as_slice_of_cells`
