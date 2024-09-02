
# Panic safety, exception safety, unwind safety

写这篇文章的起因是在给`Once<T>` 造轮子的过程中发现了一个很Trick的捕获Panic的写法，觉得非常有意思，和老板聊了之后发现这就是 Rust 中所谓的 Panic safety。既然如此，闲暇之时写一篇短文聊聊这个话题。

BTW，在各种论述之中，Exception safety, Unwind safety, Panic safety 指代的内容都是一样的。 

## Panic safety

那么何谓 Panic safety 呢？简单来说，Panic safety 是指通过 Rust 编写的代码在发生 Panic 时仍然不会带来 Undefined behavior。而更加细致的官方表述如下：

> In unsafe code, we _must_ be exception safe to the point of not violating memory safety. We'll call this _minimal_ exception safety.
> 
> In safe code, it is _good_ to be exception safe to the point of your program doing the right thing. We'll call this _maximal_ exception safety.

两层表述，Panic safety 最弱的要求是在发生 Panic的时候能够保证内存安全，而最强的要求则是在发生 Panic 的时候能够保证程序是做正确的事情。

看到这里可能有细心的读者就会问了，我们平时都是在写代码时，只有在无法处理程序状态的时候才会写 Panic，如果运行到这里，程序崩了就崩了，内存也会在进程结束的时候回收掉，也不会有什么问题嘛！

其实上述的"崩了"的表述更像是 Abort，而不是 Panic。当执行到 `panic!(...)` 这行代码的时候，**程序并不会在此时直接终止**，而是会运行一段做收尾工作的代码，也就是 Unwind。Unwind 会非常优雅的从当前 Panic 返回到上一层函数，再一层一层的向上返回，展开Backtrace，其中 **Destructor 析构器仍会工作！** Unwind 这个过程很有可能就会带来 Undefined  behavior。

在 Rust 当中，我们对于任何一个 Panic safety 的抽象，都会要求它具有一个 Invariant，即使这个抽象发生了 Panic， 也应保证这个抽象的程序逻辑和数据结构的状态是保持一致的，从而避免引发更严重的问题或数据损坏。

## 标准库 Vec 案例

标准库中的 Vec 这个抽象读者肯定都已经非常熟悉了，而它也能够保证 Panic safety。我们来看 Vec 中 `extend_with(n, value)` 这个函数，他能够让 Vec 拓宽 n 个 T 类型的内存空间，并在这段空间内逐次通过 T 的 `clone()` 方法填充`value: T`。

然而，这里的`clone()`方法就是可能发生 Panic 的地方，毕竟 Clone 是个 User-defined 函数，用户可以写出任意代码，其中就有可能包含 Panic 代码。

这里我们先忽略掉`SetLenOnDrop` 这个用于保证 Panic safety 的临时变量，后续会讲。

首先，Vec 已经通过 `reserve()` 函数将自身扩容成功，然后依次`clone()` value，填充到对应位置上。我们考虑两种情况，一是在填充前将 len 置为扩充后的真实长度，二是填充后将  len 置为扩充后的真实长度。

在第一种情况下，如果在任意一次填充时，`clone()`函数发生了 Panic，由于此时 Vec 的 len 还是之前的长度，那么剩下的内存就会被泄漏。
在第二种情况下，如果在任意一次填充时，`clone()`函数发生了 Panic，由于此时 Vec 的 len 是扩容后的长度。Vec Drop 的时候会读到剩下那些并没有被填充 value、但仍然作为 T 被析构的未初始化内存（Uninitialized memory），显然读 Uninitialized memory 也是个 Undefined  behavior。而且如果 T 有析构方法，可能会在一个 Fake 的 T 上面做析构导致奇奇怪怪的问题。

```rust
pub struct Vec<T, A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}

impl<T: Clone, A: Allocator> Vec<T, A> {
    /// Extend the vector by `n` clones of value.
    fn extend_with(&mut self, n: usize, value: T) {
        self.reserve(n);

        unsafe {
            let mut ptr = self.as_mut_ptr().add(self.len());
            // Use SetLenOnDrop to work around bug where compiler
            // might not realize the store through `ptr` through self.set_len()
            // don't alias.
            let mut local_len = SetLenOnDrop::new(&mut self.len);

            // Write all elements except the last one
            for _ in 1..n {
                ptr::write(ptr, value.clone());
                ptr = ptr.add(1);
                // Increment the length in every step in case clone() panics
                local_len.increment_len(1);
            }

            if n > 0 {
                ptr::write(ptr, value);
                local_len.increment_len(1);
            }

            // len set by scope guard
        }
    }
}
```

其实我们最希望的是，Vec 在 extend_with 中发生 Panic 时，他的 len 是真实被填充了T的 Vec 长度，可以通过 Vec 访问到这些已经初始化的内存，不会访问到那些未初始化的内存。官方 STD 标准库里面就用 RAII 的机制以一种很 trick 的方法实现了这个能力。

## 奇技淫巧

这个 `SetLenOnDrop<'a>` 就是我们要讨论的奇技淫巧，他能够捕获 Panic，并在 Unwind 过程中将 Vec len 设置为正确的值。

> 注意! Unwind 过程中会调用 Drop，析构掉 Panic 发生之前的局部变量

这里`SetLenOnDrop<'a>`是个局部变量，他包含的 len 是指向 Vec len 的一个可变引用。由于它是在可能发生 Panic 的`clone()` 调用之前被创建的，当发生了 Panic ，Unwind 会尝试去析构这个局部变量，也就会调用到 `SetLenOnDrop<'a> drop()`方法，这个时候 Unwind 会将 Vec 设置为正确的len。

```rust
pub(super) struct SetLenOnDrop<'a> {
    len: &'a mut usize,
    local_len: usize,
}

impl<'a> SetLenOnDrop<'a> {
    #[inline]
    pub(super) fn new(len: &'a mut usize) -> Self {
        SetLenOnDrop { local_len: *len, len }
    }

    #[inline]
    pub(super) fn increment_len(&mut self, increment: usize) {
        self.local_len += increment;
    }

    #[inline]
    pub(super) fn current_len(&self) -> usize {
        self.local_len
    }
}

impl Drop for SetLenOnDrop<'_> {
    #[inline]
    fn drop(&mut self) {
        *self.len = self.local_len;
    }
}
```

这个技巧非常有意思啊，由于我们知道 Unwind 会析构局部变量，所以翻转一下，我们通过局部变量来捕获 Panic，将我们想要在发生 Panic 时执行的代码逻辑写在 Drop 里，让 Panic 替我们执行一段我们希望的程序逻辑。

但是，如果我们希望这段 Panic 时执行的代码逻辑不在正常的代码执行流中，我们可以在函数正常结束时对这个局部变量调用`mem::forget()`方法，避开这段在局部变量 Drop 中执行的代码逻辑。但`mem::forget()` 了之后会不会导致这个局部变量内存被泄漏呢？答案是不会的。因为这个局部变量是在栈上分配的，当比如`extend_with()` 这个函数返回时，他的栈帧（stack frame）会被释放，rsp 寄存器上移，即使不调用 Drop 方法，也会隐性地释放掉局部变量这块内存空间。

## 总结

Panic safety 是个很强的性质，大部分情况下我们不太需要关注。正常的情况下，我们只需要使用 Result就足够了，只有极端情况下无法处理程序状态时我们才用 Panic 宏。上述的方式提供了一种捕获Panic的方法，并能够在 Panic 发生的时候完成一些事情，保证某个抽象被使用时看起来是 Invariant。

顺带一提，Rust 中也有专门用于捕获 Panic 的方法`catch_unwind`，这一方法一般用在 Rust 代码被其他语言比如 C 调用时也能够展开 Backtrace。同时 Rust 也提供了能够穿越`catch_unwind`边界的  [UnwindSafe](https://doc.rust-lang.org/std/panic/trait.UnwindSafe.html#) trait。

### 参考：
- https://rust-unofficial.github.io/too-many-lists/sixth-panics.html
- https://doc.rust-lang.org/nomicon/exception-safety.html
- https://doc.rust-lang.org/std/panic/trait.UnwindSafe.html
- https://doc.rust-lang.org/std/panic/fn.catch_unwind.html

### 拓展阅读：
- https://doc.rust-lang.org/1.80.1/src/alloc/vec/mod.rs.html#2686 (Vec中的SetLenOnDrop)
- https://docs.rs/spin/latest/src/spin/once.rs.html#252 (Once 中的Finish)
- https://doc.rust-lang.org/src/std/sync/mutex.rs.html#550 (Mutex 中的 MutexGuard poison)
