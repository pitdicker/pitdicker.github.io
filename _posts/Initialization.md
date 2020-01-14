# Initialization and `no_std`

## Regular applications

For regular applications that run on an operating system, you really want to involve the OS when one thread has to wait on an other. This is useful information for the scheduler: it would for example not make sense to pause the running thread and give the waiting thread processor time. It can decide to boost the priority of the running thread if the waiting thread has a higher priority (priority inheritance or priority ceiling protocol). And the waiting thread can be suspended so another thread can do some useful work in the meantime.

Basically, in every case where one thread has to wait on another you should always inform the OS. It is okay to spin for a few times, but if the reason to wait doesn't end soon just ask the scheduler for help. Libraries that to try to circumvent the OS by using a solution such as spin loops should be avoided.

When it comes to initialization, just use `sync::Lazy` or `Sync::OnceCell::get_or_init()`, which uses a mutex under the hood when necessary.

## Applications running on bare metal

_I don't know much about this subject, so what is written here may be inaccurate._

### Use a spin loop

If you don't have an OS, or some other kind of runtime, there is no one to inform when waiting on an other thread. Now it can be okay to use a spin loop. But make sure that same spin loop is never run inside an interrupt handler, because if the main thread was also in the critical section, we have a deadlock. One option is to temporary disable interrupts while inside the spin loop and critical section.

So for initialization you can reach for a spin loop, but it takes care to prevent deadlocks.

### Racy initialization with atomics

An alternative is to do a racy initialization. If the initialization is cheap, you can just let multiple threads do the preparations, and have only one succeed in updating an atomic variable. The other threads discard their work when done.

If the variable is small, it can be stored directly in an atomic. If the variable is larger, has a static lifetime, and if you can do allocations, put it on the heap and atomically update a pointer (see [this code](https://github.com/Amanieu/parking_lot/blob/core-0.7.0/core/src/thread_parker/windows/mod.rs#L22) in `parking_lot` as an example).

### Assume single threaded

If you know there is only ever one thread handling initialization, you can just use one atomic as guard. If the code ever changes and there is an other thread initializing concurrently, panic (or abort).

Is there a library that provides a nice API for those schemes?
- Simple spin loop, with potential deadlock in interrupt handler
- Spin loop that disables interrupts
- Racy initialization: wrap a small value in an atomic
- Racy initialization: store a large value on the heap, and put a pointer in an atomic
- Guard initilization by panicking on contention.

## Libraries

Libraries are in a difficult position. What to do if you need to initialize a global variable, and want to support both running on an OS and bare metal? If there is an OS, you should not try to circumvent it. If there is no OS, you can in no way depend on one.

What is worse, the choice can't only depend on whether `std` is available or not. A crate can be compiled without its `std` feature enabled, while there is an operating system.

1. If the standard library is always available, just use `OnceCell` or `Lazy`.
2. If initialization is cheap, consider a racy initialization scheme.
3. Delegate initialization to the user.

The last option, providing an `init` function that the user must call, seems unergonomic. On the other hand, it also seems like a small price to pay to allow using a library on bare metal.

```
impl<T> OnceCell<T> {
    #[cfg(feature = "std")]
    pub fn get_or_init_if_std<F>(&self, f: F) -> &T
        where F: FnOnce() -> T,
    {
        self.get_or_init(f)
    }
    #[cfg(not(feature = "std"))]
    pub fn get_or_init_if_std<F>(&self, f: F) -> &T
        where F: FnOnce() -> T,
    {
        self.get(f).unwrap()
    }

    try_set()
```

- delegate state storing to the user
- use atomics
- asume single threaded