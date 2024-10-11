# 예외 안전성

프로그램이 되감기를 주의해서 사용해야 하긴 하지만, `panic!`할 수 있는 코드는 많이 있습니다. `None`을 `unwrap`하거나, 범위 밖으로 인덱스를 접근하거나, 0으로 나누면 프로그램은 `panic!`할 겁니다. 
디버그 빌드에서는, 모든 수치 연산은 만약 오버플로우된다면 `panic!`합니다. 여러분이 어떤 코드가 실행되는지 매우 주의 깊게, 엄격하게 제어하지 못한다면 거의 모든 것들이 되감기를 할 수 있고, 여러분은 그것에 대비해야 합니다.

되감기에 대비하는 것은 넓은 프로그래밍 분야에서 보통 *예외 안전성이라고* 부릅니다. 러스트에서는 고려해야 하는 예외 안전성에 두 가지 단계가 있습니다:

* 불안전한 코드에서, 우리는 메모리 안전성을 깨지 않는 수준까지 예외에 안전해야 *합니다*. 이것을 *최소* 예외 안전성이라 부르겠습니다.

* 안전한 코드에서, 여러분의 프로그램이 올바른 일을 하는 수준까지 예외에 안전한 것이 *좋습니다*. 이것을 우리는 *최대* 예외 안전성이라고 하겠습니다.

러스트의 많은 부분에서 그렇듯이, 되감기에 있어서도 불안전한 코드는 나쁜 안전한 코드를 감당할 준비를 해야 합니다. 잠시 불건전한 상태를 만드는 코드는 `panic!`이 일어났을 때 그 상태가 사용되지 않도록 유의해야 합니다. 
보통 "유의한다"는 것은 이런 상태들이 존재하는 동안에는 `panic!`하지 않는 코드만 실행하도록 하거나, `panic!`이 발생할 경우 그 상태를 청소하는 안전 장치를 만드는 것입니다. 
`panic!`이 일어났을 때 항상 일관적인 상태가 아니어도 됩니다. 우리는 다만 그것이 *안전한* 상태임을 보장해야 합니다.

대부분의 불안전한 코드는 다른 코드에 연결되어 있지 않아서, 예외에 대해 안전하게 만들기 적당히 쉽습니다. 실행되는 모든 코드를 제어하고, 그 대부분은 `panic!`할 수 없죠. 
하지만 불안전한 코드에서 호출자가 제공한 코드를 반복적으로 실행하며, 일시적으로 미초기화된 데이터의 배열을 가지고 작업하는 것은 희귀한 것은 아닙니다. 이런 코드는 예외 안전성을 고려해서 주의해야 합니다.

## `Vec::push_all`

`Vec::push_all`은 어떤 슬라이스만큼 `Vec`을 확장하되 특정화 없이 비교적 효율적으로 하기 위한 임시방편입니다. 여기 간단한 구현이 있습니다:

<!-- ignore: simplified code -->
```rust,ignore
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            // 방금 이만큼 할당했기 때문에 오버플로우되지 않을 겁니다
            self.set_len(self.len() + to_push.len());

            for (i, x) in to_push.iter().enumerate() {
                self.ptr().add(i).write(x.clone());
            }
        }
    }
}
```

우리는 분명히 `Vec`이 차지할 공간을 알고 있으므로 반복되는 공간과 `len` 검사를 피하기 위해 `push`를 피해서 지나칩니다. 이 논리는 완전히 올바릅니다, 다만 물 아래 빙산 같은 한 가지 문제가 있죠: 이 코드가 예외에 안전하지 않다는 겁니다! 
`set_len`, `add` 그리고 `write`는 모두 괜찮습니다. `clone`이 바로 우리가 지나쳤던 `panic!` 시한폭탄입니다.

`Clone`은 우리의 제어를 완벽히 벗어나고, `panic!`할 자유가 차고 넘칩니다. 만약 `panic!`한다면, 우리의 함수는 `Vec`의 길이를 너무 크게 설정한 채로 일찍 종료하게 될 겁니다. 
만약 이 `Vec`을 누군가 보거나 해제하려 한다면, 미초기화된 메모리를 읽게 될 겁니다!

이 경우에 해결책은 꽤나 단순합니다. 만약 우리가 `clone`한 값들만 해제되는 것을 보장하길 원한다면, 반복문을 도는 매번 `len`을 설정하면 됩니다. 
만약 우리가 미초기화된 메모리가 관측되지 않기를 바란다면, 반복문이 끝난 후 `len`을 설정하면 됩니다.

## `BinaryHeap::sift_up`

힙에서 원소를 밀어 올리는 것은 `Vec`을 확장하는 것보다 조금 더 복잡합니다. 의사 코드는 다음과 같습니다:

```text
bubble_up(heap, index):
    while index != 0 && heap[index] < heap[parent(index)]:
        heap.swap(index, parent(index))
        index = parent(index)
```

이 의사 코드를 그대로 러스트로 옮긴다면 대체로 괜찮지만, 좀 거슬리게 성능을 방해하는 것이 있습니다: `heap[index]` 원소가 의미 없이 계속 바뀌는군요. 우리는 대신 이와 같이 할 것입니다:

```text
bubble_up(heap, index):
    let elem = heap[index]
    while index != 0 && elem < heap[parent(index)]:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

이 코드는 이제 각 원소가 최소한으로 복사된다는 것을 보장합니다 (사실 보통은 원소들이 두 번씩 복사되는 것이 필수적입니다). 하지만 이 프로그램은 이제 예외 안전성 문제가 좀 생겼습니다! 모든 순간, 어떤 값의 복사가 두 개씩 존재합니다. 
만약 이 함수에서 `panic!`한다면 무언가는 두 번 해제될 겁니다. 게다가 불행하게도, 우리는 이 코드의 완전한 제어권을ㄹ 가지고 있지도 않습니다: 비교 연산은 사용자가 정의하거든요!

`Vec`과 달리 여기서는 해결법이 쉽지는 않습니다. 한 가지 방법은 사용자가 정의한 코드와 불안전한 코드를 두 개의 분리된 단계로 나누는 것입니다:

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
