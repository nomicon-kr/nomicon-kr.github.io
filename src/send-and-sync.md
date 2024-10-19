# Send와 Sync

그래도 모든 것이 가변성을 물려받지는 않습니다. 어떤 타입들은 여러분이 메모리의 어떤 위치를 변경하면서도 여러 번 가리킬 수 있게 해 줍니다. 이 타입들이 이런 접근들을 동기화로 관리하지 않는다면 절대로 스레드 경계에서 안전하지 않습니다. 
러스트는 이런 특징을 `Send`와 `Sync` 트레잇을 통해 알아챕니다.

* 어떤 타입이 다른 스레드로 보내도 안전하다면 `Send`입니다.
* 어떤 타입이 스레드 간에 공유해도 안전하다면 `Sync`입니다 (`T`는 오직 `&T`가 `Send`인 경우에만 `Sync`입니다).

`Send`와 `Sync`는 러스트의 동시성 이야기에 근본이 됩니다. 그런 면에서, 이들이 잘 동작하도록 하는 상당한 양의 특수 장치들이 있습니다. 첫째로, 그리고 가장 중요한 것은, 이들은 [불안전한 트레잇입니다][unsafe_traits]. 
이 뜻은 이것들을 구현하는 것이 불안전하다는 의미이며, 다른 불안전한 코드가 이 구현들을 신뢰할 수 있다는 말입니다. 이것들이 *표시 트레잇이기* 때문에 (이것들은 메서드와 같은 연관된 것들이 없습니다), 
이것들이 잘 구현되었다는 것은 단지 구현하면 가져야 하는 본질적인 속성을 가졌다는 뜻입니다. `Send`나 `Sync`를 잘못 구현하는 것은 **미정의 동작을** 유발할 수 있습니다.

`Send`와 `Sync`는 또한 자동으로 파생되는 트레잇들입니다. 이것이 의미하는 것은, 다른 모든 트레잇과 달리, 어떤 타입이 `Send`/`Sync` 타입들로만 이루어져 있다면, 그 타입 또한 `Send`/`Sync`라는 것입니다. 
거의 모든 기본 타입이 `Send`이고 `Sync`이고, 그 결과로 여러분이 상호작용하게 될 거의 모든 타입들은 `Send`이고 `Sync`입니다. 

대표적인 예외들은 다음과 같습니다:

* 생 포인터들은 `Send`도 `Sync`도 아닙니다 (안전 장치가 없기 때문이죠).
* `UnsafeCell`은 `Sync`가 아닙니다 (그리고 따라서 `Cell`과 `RefCell`도 아닙니다).
* `Rc`는 `Send`도 `Sync`도 아닙니다 (참조 횟수가 공유되고, 동기화가 없기 때문입니다).

`Rc`와 `UnsafeCell`은 매우 근본적으로 스레드 경계에서 안전하지 않습니다: 동기화되지 않는 가변 공유 상태를 가지기 때문입니다. 하지만 생 포인터들은, 엄밀하게 말해서, *정보 차원에서* 스레드 경계 불안전하다고 표시된 것에 가깝습니다. 

`Rc` and `UnsafeCell` are very fundamentally not thread-safe: they enable
unsynchronized shared mutable state. However raw pointers are, strictly
speaking, marked as thread-unsafe as more of a *lint*. Doing anything useful
with a raw pointer requires dereferencing it, which is already unsafe. In that
sense, one could argue that it would be "fine" for them to be marked as thread
safe.

However it's important that they aren't thread-safe to prevent types that
contain them from being automatically marked as thread-safe. These types have
non-trivial untracked ownership, and it's unlikely that their author was
necessarily thinking hard about thread safety. In the case of `Rc`, we have a nice
example of a type that contains a `*mut` that is definitely not thread-safe.

Types that aren't automatically derived can simply implement them if desired:

```rust
struct MyBox(*mut u8);

unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}
```

In the *incredibly rare* case that a type is inappropriately automatically
derived to be Send or Sync, then one can also unimplement Send and Sync:

```rust
#![feature(negative_impls)]

// I have some magic semantics for some synchronization primitive!
struct SpecialThreadToken(u8);

impl !Send for SpecialThreadToken {}
impl !Sync for SpecialThreadToken {}
```

Note that *in and of itself* it is impossible to incorrectly derive Send and
Sync. Only types that are ascribed special meaning by other unsafe code can
possibly cause trouble by being incorrectly Send or Sync.

Most uses of raw pointers should be encapsulated behind a sufficient abstraction
that Send and Sync can be derived. For instance all of Rust's standard
collections are Send and Sync (when they contain Send and Sync types) in spite
of their pervasive use of raw pointers to manage allocations and complex ownership.
Similarly, most iterators into these collections are Send and Sync because they
largely behave like an `&` or `&mut` into the collection.

## Example

[`Box`][box-doc] is implemented as its own special intrinsic type by the
compiler for [various reasons][box-is-special], but we can implement something
with similar-ish behavior ourselves to see an example of when it is sound to
implement Send and Sync. Let's call it a `Carton`.

We start by writing code to take a value allocated on the stack and transfer it
to the heap.

```rust
# pub mod libc {
#    pub use ::std::os::raw::{c_int, c_void};
#    #[allow(non_camel_case_types)]
#    pub type size_t = usize;
#    extern "C" { pub fn posix_memalign(memptr: *mut *mut c_void, align: size_t, size: size_t) -> c_int; }
# }
use std::{
    mem::{align_of, size_of},
    ptr,
    cmp::max,
};

struct Carton<T>(ptr::NonNull<T>);

impl<T> Carton<T> {
    pub fn new(value: T) -> Self {
        // Allocate enough memory on the heap to store one T.
        assert_ne!(size_of::<T>(), 0, "Zero-sized types are out of the scope of this example");
        let mut memptr: *mut T = ptr::null_mut();
        unsafe {
            let ret = libc::posix_memalign(
                (&mut memptr as *mut *mut T).cast(),
                max(align_of::<T>(), size_of::<usize>()),
                size_of::<T>()
            );
            assert_eq!(ret, 0, "Failed to allocate or invalid alignment");
        };

        // NonNull is just a wrapper that enforces that the pointer isn't null.
        let ptr = {
            // Safety: memptr is dereferenceable because we created it from a
            // reference and have exclusive access.
            ptr::NonNull::new(memptr)
                .expect("Guaranteed non-null if posix_memalign returns 0")
        };

        // Move value from the stack to the location we allocated on the heap.
        unsafe {
            // Safety: If non-null, posix_memalign gives us a ptr that is valid
            // for writes and properly aligned.
            ptr.as_ptr().write(value);
        }

        Self(ptr)
    }
}
```

This isn't very useful, because once our users give us a value they have no way
to access it. [`Box`][box-doc] implements [`Deref`][deref-doc] and
[`DerefMut`][deref-mut-doc] so that you can access the inner value. Let's do
that.

```rust
use std::ops::{Deref, DerefMut};

# struct Carton<T>(std::ptr::NonNull<T>);
#
impl<T> Deref for Carton<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        unsafe {
            // Safety: The pointer is aligned, initialized, and dereferenceable
            //   by the logic in [`Self::new`]. We require readers to borrow the
            //   Carton, and the lifetime of the return value is elided to the
            //   lifetime of the input. This means the borrow checker will
            //   enforce that no one can mutate the contents of the Carton until
            //   the reference returned is dropped.
            self.0.as_ref()
        }
    }
}

impl<T> DerefMut for Carton<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        unsafe {
            // Safety: The pointer is aligned, initialized, and dereferenceable
            //   by the logic in [`Self::new`]. We require writers to mutably
            //   borrow the Carton, and the lifetime of the return value is
            //   elided to the lifetime of the input. This means the borrow
            //   checker will enforce that no one else can access the contents
            //   of the Carton until the mutable reference returned is dropped.
            self.0.as_mut()
        }
    }
}
```

Finally, let's think about whether our `Carton` is Send and Sync. Something can
safely be Send unless it shares mutable state with something else without
enforcing exclusive access to it. Each `Carton` has a unique pointer, so
we're good.

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
// Safety: No one besides us has the raw pointer, so we can safely transfer the
// Carton to another thread if T can be safely transferred.
unsafe impl<T> Send for Carton<T> where T: Send {}
```

What about Sync? For `Carton` to be Sync we have to enforce that you can't
write to something stored in a `&Carton` while that same something could be read
or written to from another `&Carton`. Since you need an `&mut Carton` to
write to the pointer, and the borrow checker enforces that mutable
references must be exclusive, there are no soundness issues making `Carton`
sync either.

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
// Safety: Since there exists a public way to go from a `&Carton<T>` to a `&T`
// in an unsynchronized fashion (such as `Deref`), then `Carton<T>` can't be
// `Sync` if `T` isn't.
// Conversely, `Carton` itself does not use any interior mutability whatsoever:
// all the mutations are performed through an exclusive reference (`&mut`). This
// means it suffices that `T` be `Sync` for `Carton<T>` to be `Sync`:
unsafe impl<T> Sync for Carton<T> where T: Sync  {}
```

When we assert our type is Send and Sync we usually need to enforce that every
contained type is Send and Sync. When writing custom types that behave like
standard library types we can assert that we have the same requirements.
For example, the following code asserts that a Carton is Send if the same
sort of Box would be Send, which in this case is the same as saying T is Send.

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
unsafe impl<T> Send for Carton<T> where Box<T>: Send {}
```

Right now `Carton<T>` has a memory leak, as it never frees the memory it allocates.
Once we fix that we have a new requirement we have to ensure we meet to be Send:
we need to know `free` can be called on a pointer that was yielded by an
allocation done on another thread. We can check this is true in the docs for
[`libc::free`][libc-free-docs].

```rust
# struct Carton<T>(std::ptr::NonNull<T>);
# mod libc {
#     pub use ::std::os::raw::c_void;
#     extern "C" { pub fn free(p: *mut c_void); }
# }
impl<T> Drop for Carton<T> {
    fn drop(&mut self) {
        unsafe {
            libc::free(self.0.as_ptr().cast());
        }
    }
}
```

A nice example where this does not happen is with a MutexGuard: notice how
[it is not Send][mutex-guard-not-send-docs-rs]. The implementation of MutexGuard
[uses libraries][mutex-guard-not-send-comment] that require you to ensure you
don't try to free a lock that you acquired in a different thread. If you were
able to Send a MutexGuard to another thread the destructor would run in the
thread you sent it to, violating the requirement. MutexGuard can still be Sync
because all you can send to another thread is an `&MutexGuard` and dropping a
reference does nothing.

TODO: better explain what can or can't be Send or Sync. Sufficient to appeal
only to data races?

[unsafe_traits]: safe-unsafe-meaning.html
[box-doc]: https://doc.rust-lang.org/std/boxed/struct.Box.html
[box-is-special]: https://manishearth.github.io/blog/2017/01/10/rust-tidbits-box-is-special/
[deref-doc]: https://doc.rust-lang.org/core/ops/trait.Deref.html
[deref-mut-doc]: https://doc.rust-lang.org/core/ops/trait.DerefMut.html
[mutex-guard-not-send-docs-rs]: https://doc.rust-lang.org/std/sync/struct.MutexGuard.html#impl-Send
[mutex-guard-not-send-comment]: https://github.com/rust-lang/rust/issues/23465#issuecomment-82730326
[libc-free-docs]: https://linux.die.net/man/3/free
