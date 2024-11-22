# mutex

Mutex in kernel.

Currently supported primitives:

+ Mutex: A mutual exclusion primitive.
+ mod spin: spin-locks.

Cargo Features

+ multitask: For use in the multi-threaded environments. If the feature is not enabled, Mutex will be an alias of `spin::SpinNoIrq`. This feature is enabled by default.

## Examples

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate axlog2;
extern crate alloc;

use core::panic::PanicInfo;
use axtype::{align_up_4k, align_down_4k, phys_to_virt, virt_to_phys};
use mutex::Mutex;

/// Entry
#[no_mangle]
pub extern "Rust" fn runtime_main(cpu_id: usize, dtb_pa: usize) {
    axhal::cpu::init_primary(cpu_id);

    axlog2::init("debug");
    info!("[rt_mutex]: ...");

    let start = align_up_4k(virt_to_phys(_ekernel as usize));
    let end = align_down_4k(axconfig::PHYS_MEMORY_END);
    axalloc::global_init(phys_to_virt(start), end - start);

    run_queue::init(cpu_id, dtb_pa);

    {
        let mutex: Mutex<u32> = Mutex::new(0);
        // Todo: do some tests according tests below.
        info!("{}", *mutex.lock());
    }

    info!("[rt_mutex]: ok!");
    axhal::misc::terminate();
}

#[panic_handler]
pub fn panic(info: &PanicInfo) -> ! {
    error!("{}", info);
    arch_boot::panic(info)
}

extern "C" {
    fn _ekernel();
}
```

## Structs

### `Mutex`

```rust
pub struct Mutex<T: ?Sized> { /* private fields */ }
```

A mutual exclusion primitive useful for protecting shared data, similar to std::sync::Mutex.

When the mutex is locked, the current task will block and be put into the wait queue. When the mutex is unlocked, all tasks waiting on the queue will be woken up.

#### Implementations

##### `impl<T> Mutex<T>`

```rust
pub const fn new(data: T) -> Self
```

Creates a new Mutex wrapping the supplied data.

```rust
pub fn into_inner(self) -> T
```

Consumes this Mutex and unwraps the underlying data.

##### `impl<T: ?Sized> Mutex<T>`

```rust
pub fn is_locked(&self) -> bool
```

Returns true if the lock is currently held.

**Safety**
This function provides no synchronization guarantees and so its result should be considered ‘out of date’ the instant it is called. Do not use it for synchronization purposes. However, it may be useful as a heuristic.

```rust
pub fn lock(&self) -> MutexGuard<'_, T>
```

Locks the Mutex and returns a guard that permits access to the inner data.

The returned value may be dereferenced for data access and the lock will be dropped when the guard falls out of scope.

```rust
pub fn try_lock(&self) -> Option<MutexGuard<'_, T>>
```

Try to lock this Mutex, returning a lock guard if successful.

```rust
pub unsafe fn force_unlock(&self)
```

Force unlock the Mutex.

**Safety**
This is extremely unsafe if the lock is not held by the current thread. However, this can be useful in some instances for exposing the lock to FFI that doesn’t know how to deal with RAII.

```rust
pub fn get_mut(&mut self) -> &mut T
```

Returns a mutable reference to the underlying data.

Since this call borrows the Mutex mutably, and a mutable reference is guaranteed to be exclusive in Rust, no actual locking needs to take place – the mutable borrow statically guarantees no locks exist. As such, this is a ‘zero-cost’ operation.

### `MutexGuard`

```rust
pub struct MutexGuard<'a, T: ?Sized + 'a> { /* private fields */ }
```

A guard that provides mutable data access.
When the guard falls out of scope it will release the lock.

#### Trait Implementations

##### `impl<'a, T: ?Sized + Debug> Debug for MutexGuard<'a, T>`

```rust
fn fmt(&self, f: &mut Formatter<'_>) -> Result
```

Formats the value using the given formatter. Read more

##### `impl<'a, T: ?Sized> Deref for MutexGuard<'a, T>`

```rust
type Target = T
```

The resulting type after dereferencing.

```rust
fn deref(&self) -> &T
```

Dereferences the value.

##### `impl<'a, T: ?Sized> DerefMut for MutexGuard<'a, T>`

```rust
fn deref_mut(&mut self) -> &mut T
```

Mutably dereferences the value.

##### `impl<'a, T: ?Sized> Drop for MutexGuard<'a, T>`

```rust
fn drop(&mut self)
```

The dropping of the MutexGuard will release the lock it was created from.
