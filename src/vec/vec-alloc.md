# 메모리 할당하기

`NonNull`을 사용하는 것은 `Vec`(과 모든 표준 컬렉션들)의 중요한 동작에 걸림돌이 됩니다: 비어있는 `Vec`을 생성하면 실제로는 아무것도 할당하지 않아야 한다는 것입니다. 이것은 크기가 0인 메모리를 할당하는 것과는 다릅니다, 전역 할당자에게는 크기가 0인 메모리를 할당하는 것은 용납되지 않거든요 (이렇게 하면 미정의 동작이 일어납니다!). 따라서 우리가 할당할 수도 없고, `ptr`에 널 포인터를 넣을 수도 없다면, `Vec::new`에서는 어떻게 하죠? 음, 그냥 아무 쓰레기 값이나 집어넣죠!

할당이 없을 때 `cap == 0`을 먼저 확인하도록 할 것이기 때문에 이렇게 해도 전혀 문제가 없습니다. 심지어 우리는 이 경우를 특별히 처리할 필요도 거의 없는데, 어차피 보통 우리는 `cap > len`이나 `len > 0`을 확인해야 하기 때문입니다. 여기에 들어가기에 추천되는 러스트 값은 `mem::align_of::<T>()`입니다. `NonNull`에서는 이것을 위한 편리 함수를 제공합니다: `NonNull::dangling()`입니다. 우리는 꽤 많은 곳에서 `dangling`을 사용할 텐데, 실제 할당은 없지만 `null`은 컴파일러가 나쁜 짓을 하도록 만들 것이기 때문입니다.

그래서:

<!-- ignore: explanation code -->
```rust,ignore
use std::mem;

impl<T> Vec<T> {
    pub fn new() -> Self {
        assert!(mem::size_of::<T>() != 0, "아직 영량 타입을 처리할 준비가 되지 않았습니다!");
        Vec {
            ptr: NonNull::dangling(),
            len: 0,
            cap: 0,
        }
    }
}
# fn main() {}
```

저 코드에 `assert` 문이 들어가 있는데, 영량 타입은 우리의 코드에서 전반적으로 특별한 처리가 필요하고, 그 문제를 지금은 보류하고 싶기 때문입니다. 이 `assert` 문이 없으면, 우리의 초기 코드는 **매우 나쁜 짓을** 할 겁니다.

다음으로 우리는 *실제로* 공간을 원할 때 무엇을 해야 할지를 고민해야 합니다. 이것을 위해서 우리는 안정적인 러스트의 [`std::alloc`에][std_alloc] 있는 전역 할당 함수 [`alloc`][alloc], [`realloc`][realloc],
그리고 [`dealloc`을][dealloc] 사용합니다. 이 함수들은 나중에 [`std::alloc::Global`][Global] 타입이 표준화되고 나면 구식으로 취급될 겁니다.

우리는 또한 메모리 소진 상황(out-of-memory, OOM)도 처리해야 합니다. 표준 라이브러리는 [`alloc::handle_alloc_error`][handle_alloc_error] 함수를 제공하는데, 이 함수는 플랫폼마다 다른 방식으로 프로그램을 강제 종료합니다. 우리가 `panic!`하지 않고 강제 종료하는 이유는 되감기 작업에서 할당이 일어날 수 있는데, 할당자가 "앗, 메모리가 더 이상 없습니다"라고 말하며 돌아왔는데 할당이 일어나는 것은 나쁜 일 같기 때문입니다.

물론, 대부분의 플랫폼은 일반적인 상황에서는 메모리가 소진되는 일이 없기 때문에, 이렇게 처리하는 것은 좀 바보같아 보입니다. 여러분이 실제로 모든 메모리를 다 사용하기 시작할 때 여러분의 운영 체제는 아마 다른 수단을 동원해서 그 프로그램을 종료시킬 겁니다. 우리가 OOM을 초래할 가장 가능성 있는 방법은 터무니없을 정도로 많은 양의 메모리를 한번에 요청하는 것입니다 (예: 이론적인 주소 공간의 절반). 따라서 *아마도* `panic!`해도 괜찮고, 나쁜 일은 일어나지 않을 겁니다. 그래도 우리는 최대한 표준 라이브러리와 같이 되도록 노력하는 것이기 때문에, 우리는 그냥 전체 프로그램을 강제 종료할 겁니다.

좋습니다, 이제 우리는 성장하는 논리를 작성할 수 있겠군요, 대략적으로 우리는 이런 논리를 원합니다:

```text
if cap == 0:
    allocate()
    cap = 1
else:
    reallocate()
    cap *= 2
```

But Rust's only supported allocator API is so low level that we'll need to do a
fair bit of extra work. We also need to guard against some special
conditions that can occur with really large allocations or empty allocations.

In particular, `ptr::offset` will cause us a lot of trouble, because it has
the semantics of LLVM's GEP inbounds instruction. If you're fortunate enough to
not have dealt with this instruction, here's the basic story with GEP: alias
analysis, alias analysis, alias analysis. It's super important to an optimizing
compiler to be able to reason about data dependencies and aliasing.

As a simple example, consider the following fragment of code:

<!-- ignore: simplified code -->
```rust,ignore
*x *= 7;
*y *= 3;
```

If the compiler can prove that `x` and `y` point to different locations in
memory, the two operations can in theory be executed in parallel (by e.g.
loading them into different registers and working on them independently).
However the compiler can't do this in general because if x and y point to
the same location in memory, the operations need to be done to the same value,
and they can't just be merged afterwards.

When you use GEP inbounds, you are specifically telling LLVM that the offsets
you're about to do are within the bounds of a single "allocated" entity. The
ultimate payoff being that LLVM can assume that if two pointers are known to
point to two disjoint objects, all the offsets of those pointers are *also*
known to not alias (because you won't just end up in some random place in
memory). LLVM is heavily optimized to work with GEP offsets, and inbounds
offsets are the best of all, so it's important that we use them as much as
possible.

So that's what GEP's about, how can it cause us trouble?

The first problem is that we index into arrays with unsigned integers, but
GEP (and as a consequence `ptr::offset`) takes a signed integer. This means
that half of the seemingly valid indices into an array will overflow GEP and
actually go in the wrong direction! As such we must limit all allocations to
`isize::MAX` elements. This actually means we only need to worry about
byte-sized objects, because e.g. `> isize::MAX` `u16`s will truly exhaust all of
the system's memory. However in order to avoid subtle corner cases where someone
reinterprets some array of `< isize::MAX` objects as bytes, std limits all
allocations to `isize::MAX` bytes.

On all 64-bit targets that Rust currently supports we're artificially limited
to significantly less than all 64 bits of the address space (modern x64
platforms only expose 48-bit addressing), so we can rely on just running out of
memory first. However on 32-bit targets, particularly those with extensions to
use more of the address space (PAE x86 or x32), it's theoretically possible to
successfully allocate more than `isize::MAX` bytes of memory.

However since this is a tutorial, we're not going to be particularly optimal
here, and just unconditionally check, rather than use clever platform-specific
`cfg`s.

The other corner-case we need to worry about is empty allocations. There will
be two kinds of empty allocations we need to worry about: `cap = 0` for all T,
and `cap > 0` for zero-sized types.

These cases are tricky because they come
down to what LLVM means by "allocated". LLVM's notion of an
allocation is significantly more abstract than how we usually use it. Because
LLVM needs to work with different languages' semantics and custom allocators,
it can't really intimately understand allocation. Instead, the main idea behind
allocation is "doesn't overlap with other stuff". That is, heap allocations,
stack allocations, and globals don't randomly overlap. Yep, it's about alias
analysis. As such, Rust can technically play a bit fast and loose with the notion of
an allocation as long as it's *consistent*.

Getting back to the empty allocation case, there are a couple of places where
we want to offset by 0 as a consequence of generic code. The question is then:
is it consistent to do so? For zero-sized types, we have concluded that it is
indeed consistent to do a GEP inbounds offset by an arbitrary number of
elements. This is a runtime no-op because every element takes up no space,
and it's fine to pretend that there's infinite zero-sized types allocated
at `0x01`. No allocator will ever allocate that address, because they won't
allocate `0x00` and they generally allocate to some minimal alignment higher
than a byte. Also generally the whole first page of memory is
protected from being allocated anyway (a whole 4k, on many platforms).

However what about for positive-sized types? That one's a bit trickier. In
principle, you can argue that offsetting by 0 gives LLVM no information: either
there's an element before the address or after it, but it can't know which.
However we've chosen to conservatively assume that it may do bad things. As
such we will guard against this case explicitly.

*Phew*

Ok with all the nonsense out of the way, let's actually allocate some memory:

<!-- ignore: simplified code -->
```rust,ignore
use std::alloc::{self, Layout};

impl<T> Vec<T> {
    fn grow(&mut self) {
        let (new_cap, new_layout) = if self.cap == 0 {
            (1, Layout::array::<T>(1).unwrap())
        } else {
            // This can't overflow since self.cap <= isize::MAX.
            let new_cap = 2 * self.cap;

            // `Layout::array` checks that the number of bytes is <= usize::MAX,
            // but this is redundant since old_layout.size() <= isize::MAX,
            // so the `unwrap` should never fail.
            let new_layout = Layout::array::<T>(new_cap).unwrap();
            (new_cap, new_layout)
        };

        // Ensure that the new allocation doesn't exceed `isize::MAX` bytes.
        assert!(new_layout.size() <= isize::MAX as usize, "Allocation too large");

        let new_ptr = if self.cap == 0 {
            unsafe { alloc::alloc(new_layout) }
        } else {
            let old_layout = Layout::array::<T>(self.cap).unwrap();
            let old_ptr = self.ptr.as_ptr() as *mut u8;
            unsafe { alloc::realloc(old_ptr, old_layout, new_layout.size()) }
        };

        // If allocation fails, `new_ptr` will be null, in which case we abort.
        self.ptr = match NonNull::new(new_ptr as *mut T) {
            Some(p) => p,
            None => alloc::handle_alloc_error(new_layout),
        };
        self.cap = new_cap;
    }
}
# fn main() {}
```

[Global]: ../../std/alloc/struct.Global.html
[handle_alloc_error]: ../../alloc/alloc/fn.handle_alloc_error.html
[alloc]: ../../alloc/alloc/fn.alloc.html
[realloc]: ../../alloc/alloc/fn.realloc.html
[dealloc]: ../../alloc/alloc/fn.dealloc.html
[std_alloc]: ../../alloc/alloc/index.html
