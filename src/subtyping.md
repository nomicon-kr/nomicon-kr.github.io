# 부분타입 다형성과 변성(變性, Variance)

러스트는 빌림과 소유권 사이의 관계를 추적하기 위해 수명을 사용합니다. 하지만 수명의 순진한 구현은 너무 제한적이거나, 아니면 미정의 동작을 허용하게 됩니다.

수명을 유연하게 사용하면서도 수명의 오용을 방지하기 위해서, 러스트는 **부분타입 다형성** 과 **변성(變性, Variance)** 을 사용합니다.

예제와 함께 시작해 보죠.

```rust
// 주의: debug는 수명이 *같은* 두 개의 매개변수를 기대합니다.
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world 는 'static 보다 짧은 수명입니다
        debug(hello, world);
    }
}
```

보수적인 수명의 구현에서는 `hello`와 `world`는 다른 수명을 가지고 있으므로, 우리는 다음과 같은 오류를 볼지도 모릅니다:

```text
error[E0308]: mismatched types
 --> src/main.rs:10:16
   |
10 |         debug(hello, world);
   |                      ^
   |                      |
   |                      expected `&'static str`, found struct `&'world str`
```

이것은 뭔가 부적절할 것입니다. 이 경우에 우리가 원하는 것은 *최소한* `'world`만큼만 사는 타입은 모두 받는 것입니다. 우리의 수명들에 부분타입 다형성을 이용해 봅시다.

## 부분타입 다형성

부분타입 다형성은 한 타입이 다른 타입 대신에 쓰일 수 있다는 개념입니다.

`Sub`이라는 타입이 `Super`라는 타입의 부분타입이라고 해 봅시다 (우리는 이 단원에서 이것을 `Sub <: Super`라고 표현하는 표기법을 사용하겠습니다).

이것이 우리에게 나타내는 것은 `Super`가 정의하는 *요구사항들*의 집합을 `Sub`이 완벽하게 충족한다는 것입니다. 그 다음 `Sub`은 더 많은 요구사항을 가질 수 있겠죠.

이제, 부분타입 다형성을 수명에 쓰기 위해, 우리는 수명의 요구사항을 정의해야 합니다:

> `'a`는 코드 구역을 정의한다.

이제 수명을 위한 요구사항을 만들었으니, 우리는 수명들이 서로 어떻게 관련이 있는지를 정의할 수 있습니다:

> `'long`이 정의하는 코드 구역이 `'short`가 정의하는 구역을 **완전히 포함할 때**, 그리고 오직 그 경우에만 `'long <: 'short`이다.



`'long` may define a region larger than `'short`, but that still fits our definition.

> As we will see throughout the rest of this chapter,
subtyping is a lot more complicated and subtle than this,
but this simple rule is a very good 99% intuition.
And unless you write unsafe code, the compiler will automatically handle all the corner cases for you.

> But this is the Rustonomicon. We're writing unsafe code,
so we need to understand how this stuff really works, and how we can mess it up.

Going back to our example above, we can say that `'static <: 'world`.
For now, let's also accept the idea that subtypes of lifetimes can be passed through references
(more on this in [Variance](#variance)),
_e.g._ `&'static str` is a subtype of `&'world str`, then we can "downgrade" `&'static str` into a `&'world str`.
With that, the example above will compile:

```rust
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let hello: &'static str = "hello";
    {
        let world = String::from("world");
        let world = &world; // 'world has a shorter lifetime than 'static
        debug(hello, world); // hello silently downgrades from `&'static str` into `&'world str`
    }
}
```

## Variance

Above, we glossed over the fact that `'static <: 'b` implied that `&'static T <: &'b T`. This uses a property known as _variance_.
It's not always as simple as this example, though. To understand that, let's try to extend this example a bit:

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}"); // use after free 😿
}
```

In `assign`, we are setting the `hello` reference to point to `world`.
But then `world` goes out of scope, before the later use of `hello` in the println!

This is a classic use-after-free bug!

Our first instinct might be to blame the `assign` impl, but there's really nothing wrong here.
It shouldn't be surprising that we might want to assign a `T` into a `T`.

The problem is that we cannot assume that `&mut &'static str` and `&mut &'b str` are compatible.
This means that `&mut &'static str` **cannot** be a *subtype* of `&mut &'b str`,
even if `'static` is a subtype of `'b`.

Variance is the concept that Rust borrows to define relationships about subtypes through their generic parameters.

> NOTE: For convenience we will define a generic type `F<T>` so
> that we can easily talk about `T`. Hopefully this is clear in context.

The type `F`'s *variance* is how the subtyping of its inputs affects the
subtyping of its outputs. There are three kinds of variance in Rust. Given two
types `Sub` and `Super`, where `Sub` is a subtype of `Super`:

* `F` is **covariant** if `F<Sub>` is a subtype of `F<Super>` (the subtype property is passed through)
* `F` is **contravariant** if `F<Super>` is a subtype of `F<Sub>` (the subtype property is "inverted")
* `F` is **invariant** otherwise (no subtyping relationship exists)

If we remember from the above examples,
it was ok for us to treat `&'a T` as a subtype of `&'b T` if `'a <: 'b`,
therefore we can say that `&'a T` is *covariant* over `'a`.

Also, we saw that it was not ok for us to treat `&mut &'a U` as a subtype of `&mut &'b U`,
therefore we can say that `&mut T` is *invariant* over `T`

Here is a table of some other generic types and their variances:

|                 |     'a    |         T         |     U     |
|-----------------|:---------:|:-----------------:|:---------:|
| `&'a T `        | covariant | covariant         |           |
| `&'a mut T`     | covariant | invariant         |           |
| `Box<T>`        |           | covariant         |           |
| `Vec<T>`        |           | covariant         |           |
| `UnsafeCell<T>` |           | invariant         |           |
| `Cell<T>`       |           | invariant         |           |
| `fn(T) -> U`    |           | **contra**variant | covariant |
| `*const T`      |           | covariant         |           |
| `*mut T`        |           | invariant         |           |

Some of these can be explained simply in relation to the others:

* `Vec<T>` and all other owning pointers and collections follow the same logic as `Box<T>`
* `Cell<T>` and all other interior mutability types follow the same logic as `UnsafeCell<T>`
* `UnsafeCell<T>` having interior mutability gives it the same variance properties as `&mut T`
* `*const T` follows the logic of `&T`
* `*mut T` follows the logic of `&mut T` (or `UnsafeCell<T>`)

For more types, see the ["Variance" section][variance-table] on the reference.

[variance-table]: ../reference/subtyping.html#variance

> NOTE: the *only* source of contravariance in the language is the arguments to
> a function, which is why it really doesn't come up much in practice. Invoking
> contravariance involves higher-order programming with function pointers that
> take references with specific lifetimes (as opposed to the usual "any lifetime",
> which gets into higher rank lifetimes, which work independently of subtyping).

Now that we have some more formal understanding of variance,
let's go through some more examples in more detail.

```rust,compile_fail,E0597
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}

fn main() {
    let mut hello: &'static str = "hello";
    {
        let world = String::from("world");
        assign(&mut hello, &world);
    }
    println!("{hello}");
}
```

And what do we get when we run this?

```text
error[E0597]: `world` does not live long enough
  --> src/main.rs:9:28
   |
6  |     let mut hello: &'static str = "hello";
   |                    ------------ type annotation requires that `world` is borrowed for `'static`
...
9  |         assign(&mut hello, &world);
   |                            ^^^^^^ borrowed value does not live long enough
10 |     }
   |     - `world` dropped here while still borrowed
```

Good, it doesn't compile! Let's break down what's happening here in detail.

First let's look at the `assign` function:

```rust
fn assign<T>(input: &mut T, val: T) {
    *input = val;
}
```

All it does is take a mutable reference and a value and overwrite the referent with it.
What's important about this function is that it creates a type equality constraint. It
clearly says in its signature the referent and the value must be the *exact same* type.

Meanwhile, in the caller we pass in `&mut &'static str` and `&'world str`.

Because `&mut T` is invariant over `T`, the compiler concludes it can't apply any subtyping
to the first argument, and so `T` must be exactly `&'static str`.

This is counter to the `&T` case:

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {a:?} b = {b:?}");
}
```

where similarly `a` and `b` must have the same type `T`.
But since `&'a T` *is* covariant over `'a`, we are allowed to perform subtyping.
So the compiler decides that `&'static str` can become `&'b str` if and only if
`&'static str` is a subtype of `&'b str`, which will hold if `'static <: 'b`.
This is true, so the compiler is happy to continue compiling this code.

As it turns out, the argument for why it's ok for Box (and Vec, HashMap, etc.) to be covariant is pretty similar to the argument for why it's ok for lifetimes to be covariant: as soon as you try to stuff them in something like a mutable reference, they inherit invariance and you're prevented from doing anything bad.

However Box makes it easier to focus on the by-value aspect of references that we partially glossed over.

Unlike a lot of languages which allow values to be freely aliased at all times, Rust has a very strict rule: if you're allowed to mutate or move a value, you are guaranteed to be the only one with access to it.

Consider the following code:

```rust,ignore
let hello: Box<&'static str> = Box::new("hello");

let mut world: Box<&'b str>;
world = hello;
```

There is no problem at all with the fact that we have forgotten that `hello` was alive for `'static`,
because as soon as we moved `hello` to a variable that only knew it was alive for `'b`,
**we destroyed the only thing in the universe that remembered it lived for longer**!

Only one thing left to explain: function pointers.

To see why `fn(T) -> U` should be covariant over `U`, consider the following signature:

<!-- ignore: simplified code -->
```rust,ignore
fn get_str() -> &'a str;
```

This function claims to produce a `str` bound by some lifetime `'a`. As such, it is perfectly valid to
provide a function with the following signature instead:

<!-- ignore: simplified code -->
```rust,ignore
fn get_static() -> &'static str;
```

So when the function is called, all it's expecting is a `&str` which lives at least the lifetime of `'a`,
it doesn't matter if the value actually lives longer.

However, the same logic does not apply to *arguments*. Consider trying to satisfy:

<!-- ignore: simplified code -->
```rust,ignore
fn store_ref(&'a str);
```

with:

<!-- ignore: simplified code -->
```rust,ignore
fn store_static(&'static str);
```

The first function can accept any string reference as long as it lives at least for `'a`,
but the second cannot accept a string reference that lives for any duration less than `'static`,
which would cause a conflict.
Covariance doesn't work here. But if we flip it around, it actually *does*
work! If we need a function that can handle `&'static str`, a function that can handle *any* reference lifetime
will surely work fine.

Let's see this in practice

```rust,compile_fail
# use std::cell::RefCell;
thread_local! {
    pub static StaticVecs: RefCell<Vec<&'static str>> = RefCell::new(Vec::new());
}

/// saves the input given into a thread local `Vec<&'static str>`
fn store(input: &'static str) {
    StaticVecs.with_borrow_mut(|v| v.push(input));
}

/// Calls the function with it's input (must have the same lifetime!)
fn demo<'a>(input: &'a str, f: fn(&'a str)) {
    f(input);
}

fn main() {
    demo("hello", store); // "hello" is 'static. Can call `store` fine

    {
        let smuggle = String::from("smuggle");

        // `&smuggle` is not static. If we were to call `store` with `&smuggle`,
        // we would have pushed an invalid lifetime into the `StaticVecs`.
        // Therefore, `fn(&'static str)` cannot be a subtype of `fn(&'a str)`
        demo(&smuggle, store);
    }

    // use after free 😿
    StaticVecs.with_borrow(|v| println!("{v:?}"));
}
```

And that's why function types, unlike anything else in the language, are
**contra**variant over their arguments.

Now, this is all well and good for the types the standard library provides, but
how is variance determined for types that *you* define? A struct, informally
speaking, inherits the variance of its fields. If a struct `MyType`
has a generic argument `A` that is used in a field `a`, then MyType's variance
over `A` is exactly `a`'s variance over `A`.

However if `A` is used in multiple fields:

* If all uses of `A` are covariant, then MyType is covariant over `A`
* If all uses of `A` are contravariant, then MyType is contravariant over `A`
* Otherwise, MyType is invariant over `A`

```rust
use std::cell::Cell;

struct MyType<'a, 'b, A: 'a, B: 'b, C, D, E, F, G, H, In, Out, Mixed> {
    a: &'a A,     // covariant over 'a and A
    b: &'b mut B, // covariant over 'b and invariant over B

    c: *const C,  // covariant over C
    d: *mut D,    // invariant over D

    e: E,         // covariant over E
    f: Vec<F>,    // covariant over F
    g: Cell<G>,   // invariant over G

    h1: H,        // would also be covariant over H except...
    h2: Cell<H>,  // invariant over H, because invariance wins all conflicts

    i: fn(In) -> Out,       // contravariant over In, covariant over Out

    k1: fn(Mixed) -> usize, // would be contravariant over Mixed except..
    k2: Mixed,              // invariant over Mixed, because invariance wins all conflicts
}
```
