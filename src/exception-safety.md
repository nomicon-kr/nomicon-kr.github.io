# 예외 안전성

프로그램이 되감기를 주의해서 사용해야 하긴 하지만, `panic!`할 수 있는 코드는 많이 있습니다. `None`을 `unwrap`하거나, 범위 밖으로 인덱스를 접근하거나, 0으로 나누면 프로그램은 `panic!`할 겁니다. 
디버그 빌드에서는, 모든 수치 연산은 만약 오버플로우된다면 `panic!`합니다. 여러분이 어떤 코드가 실행되는지 매우 주의 깊게, 엄격하게 제어하지 못한다면 거의 모든 것들이 되감기를 할 수 있고, 여러분은 그것에 대비해야 합니다.

되감기에 대비하는 것은 넓은 프로그래밍 분야에서 보통 *예외 안전성이라고* 부릅니다. 러스트에서는 고려해야 하는 예외 안전성에 두 가지 단계가 있습니다:

* 불안전한 코드에서, 우리는 메모리 안전성을 깨지 않는 수준까지 예외에 안전해야 *합니다*. 이것을 *최소* 예외 안전성이라 부르겠습니다.

* 안전한 코드에서, 여러분의 프로그램이 올바른 일을 하는 수준까지 예외에 안전한 것이 *좋습니다*. 이것을 우리는 *최대* 예외 안전성이라고 하겠습니다.

러스트의 많은 부분에서 그렇듯이, 되감기에 있어서도 불안전한 코드는 나쁜 안전한 코드를 감당할 준비를 해야 합니다. 잠시 불건전한 상태를 만드는 코드는 `panic!`이 일어났을 때 그 상태가 사용되지 않도록 유의해야 합니다. 
보통 "유의한다"는 것은 이런 상태들이 존재하는 동안에는 `panic!`하지 않는 코드만 실행하도록 하거나, `panic!`이 발생할 경우 그 상태를 청소하는 안전 장치를 만드는 것입니다. 
`panic!`이 일어났을 때 항상 일관적인 상태가 아니어도 됩니다. 우리는 다만 그것이 *안전한* 상태임을 보장해야 합니다.



Most Unsafe code is leaf-like, and therefore fairly easy to make exception-safe.
It controls all the code that runs, and most of that code can't panic. However
it is not uncommon for Unsafe code to work with arrays of temporarily
uninitialized data while repeatedly invoking caller-provided code. Such code
needs to be careful and consider exception safety.

## Vec::push_all

`Vec::push_all` is a temporary hack to get extending a Vec by a slice reliably
efficient without specialization. Here's a simple implementation:

<!-- ignore: simplified code -->
```rust,ignore
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            // can't overflow because we just reserved this
            self.set_len(self.len() + to_push.len());

            for (i, x) in to_push.iter().enumerate() {
                self.ptr().add(i).write(x.clone());
            }
        }
    }
}
```

We bypass `push` in order to avoid redundant capacity and `len` checks on the
Vec that we definitely know has capacity. The logic is totally correct, except
there's a subtle problem with our code: it's not exception-safe! `set_len`,
`add`, and `write` are all fine; `clone` is the panic bomb we over-looked.

Clone is completely out of our control, and is totally free to panic. If it
does, our function will exit early with the length of the Vec set too large. If
the Vec is looked at or dropped, uninitialized memory will be read!

The fix in this case is fairly simple. If we want to guarantee that the values
we *did* clone are dropped, we can set the `len` every loop iteration. If we
just want to guarantee that uninitialized memory can't be observed, we can set
the `len` after the loop.

## BinaryHeap::sift_up

Bubbling an element up a heap is a bit more complicated than extending a Vec.
The pseudocode is as follows:

```text
bubble_up(heap, index):
    while index != 0 && heap[index] < heap[parent(index)]:
        heap.swap(index, parent(index))
        index = parent(index)
```

A literal transcription of this code to Rust is totally fine, but has an annoying
performance characteristic: the `self` element is swapped over and over again
uselessly. We would rather have the following:

```text
bubble_up(heap, index):
    let elem = heap[index]
    while index != 0 && elem < heap[parent(index)]:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

This code ensures that each element is copied as little as possible (it is in
fact necessary that elem be copied twice in general). However it now exposes
some exception safety trouble! At all times, there exists two copies of one
value. If we panic in this function something will be double-dropped.
Unfortunately, we also don't have full control of the code: that comparison is
user-defined!

Unlike Vec, the fix isn't as easy here. One option is to break the user-defined
code and the unsafe code into two separate phases:

```text
bubble_up(heap, index):
    let end_index = index;
    while end_index != 0 && heap[end_index] < heap[parent(end_index)]:
        end_index = parent(end_index)

    let elem = heap[index]
    while index != end_index:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

If the user-defined code blows up, that's no problem anymore, because we haven't
actually touched the state of the heap yet. Once we do start messing with the
heap, we're working with only data and functions that we trust, so there's no
concern of panics.

Perhaps you're not happy with this design. Surely it's cheating! And we have
to do the complex heap traversal *twice*! Alright, let's bite the bullet. Let's
intermix untrusted and unsafe code *for reals*.

If Rust had `try` and `finally` like in Java, we could do the following:

```text
bubble_up(heap, index):
    let elem = heap[index]
    try:
        while index != 0 && elem < heap[parent(index)]:
            heap[index] = heap[parent(index)]
            index = parent(index)
    finally:
        heap[index] = elem
```

The basic idea is simple: if the comparison panics, we just toss the loose
element in the logically uninitialized index and bail out. Anyone who observes
the heap will see a potentially *inconsistent* heap, but at least it won't
cause any double-drops! If the algorithm terminates normally, then this
operation happens to coincide precisely with how we finish up regardless.

Sadly, Rust has no such construct, so we're going to need to roll our own! The
way to do this is to store the algorithm's state in a separate struct with a
destructor for the "finally" logic. Whether we panic or not, that destructor
will run and clean up after us.

<!-- ignore: simplified code -->
```rust,ignore
struct Hole<'a, T: 'a> {
    data: &'a mut [T],
    /// `elt` is always `Some` from new until drop.
    elt: Option<T>,
    pos: usize,
}

impl<'a, T> Hole<'a, T> {
    fn new(data: &'a mut [T], pos: usize) -> Self {
        unsafe {
            let elt = ptr::read(&data[pos]);
            Hole {
                data,
                elt: Some(elt),
                pos,
            }
        }
    }

    fn pos(&self) -> usize { self.pos }

    fn removed(&self) -> &T { self.elt.as_ref().unwrap() }

    fn get(&self, index: usize) -> &T { &self.data[index] }

    unsafe fn move_to(&mut self, index: usize) {
        let index_ptr: *const _ = &self.data[index];
        let hole_ptr = &mut self.data[self.pos];
        ptr::copy_nonoverlapping(index_ptr, hole_ptr, 1);
        self.pos = index;
    }
}

impl<'a, T> Drop for Hole<'a, T> {
    fn drop(&mut self) {
        // fill the hole again
        unsafe {
            let pos = self.pos;
            ptr::write(&mut self.data[pos], self.elt.take().unwrap());
        }
    }
}

impl<T: Ord> BinaryHeap<T> {
    fn sift_up(&mut self, pos: usize) {
        unsafe {
            // Take out the value at `pos` and create a hole.
            let mut hole = Hole::new(&mut self.data, pos);

            while hole.pos() != 0 {
                let parent = parent(hole.pos());
                if hole.removed() <= hole.get(parent) { break }
                hole.move_to(parent);
            }
            // Hole will be unconditionally filled here; panic or not!
        }
    }
}
```
