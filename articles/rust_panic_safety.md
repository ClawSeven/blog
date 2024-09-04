# Rust 中如何保证 Panic safety

写这篇文章的起因是在给`Once<T>` 造轮子的过程中发现了一个很Trick的捕获Panic的写法，觉得非常有意思，和老板聊了之后发现这就是 Rust 中所谓的 Panic safety。既然如此，闲暇之时写一篇短文聊聊这个话题。在各种论述之中，Exception safety, Unwind safety, Panic safety 指代的内容都是一样的。 

> 内存安全架构原则：  
> 1.隔离并模块化由非内存安全代码编写的组件，并最小化其代码量；  
> 2.由非内存安全代码编写的组件不应减弱安全模块的安全性，尤其是公共 API 和公共数据结构；
> 3.由非内存安全代码编写的组件需清晰可辨识并且易于更新。  
>              --Lenx

## Panic safety 是什么

那么何谓 Panic safety 呢？简单来说，Panic safety 是指通过 Rust 编写的代码在发生 Panic 时仍然不会带来 Undefined behavior。而更加细致的官方表述如下：

> In unsafe code, we _must_ be exception safe to the point of not violating memory safety. We'll call this _minimal_ exception safety.
> 
> In safe code, it is _good_ to be exception safe to the point of your program doing the right thing. We'll call this _maximal_ exception safety.

两层表述，Panic safety 最弱的要求是在发生 Panic的时候能够保证内存安全，而最强的要求则是在发生 Panic 的时候能够保证程序是做正确的事情。

看到这里可能有细心的读者就会问了，我们平时都是在写代码时，只有在无法处理程序状态的时候才会写 Panic，如果运行到这里，程序崩了就崩了，内存也会在进程结束的时候回收掉，也不会有什么问题嘛！

其实上述的"崩了"的表述更像是 Abort，而不是 Panic。当执行到 `panic!(...)` 这行代码的时候，**程序并不会在此时直接终止**，而是会运行一段做收尾工作的代码，也就是 Unwind。Unwind 会非常优雅的从当前 Panic 返回到上一层函数，再一层一层的向上返回，展开Backtrace，其中 **Destructor 析构器仍会工作！** Unwind 这个过程很有可能就会带来 Undefined  behavior。

Rust 对于 Unwind 过程中带来 Undefined behavior 也有比较清晰的论述，当只有满足以下两点时才会出现问题：

> 1. A data structure is in a temporarily invalid state when the thread panics.
> 2. This broken invariant is then later observed.

其一，发生 Panic 时数据结构处在一个无效状态。其二，这个被破坏的数据结构（某个不变属性）随后被观测了。在 Rust 当中，我们对于任何一个 Panic safety 的抽象，都会要求它具有一个 Invariant，即使这个抽象发生了 Panic， 也应保证这个抽象的程序逻辑和数据结构的状态是保持一致的，从而避免引发更严重的问题或数据损坏。

## Panic 过程简述

这一节我们介绍一下 Panic 过程中，运行时都做了什么事情。先设定前提，这是在 std 环境下。当 Panic 发生时，会先进行一次双重 Panic 检测，看是否在 Panic 处理过程中也发生了 Panic。如果发生了双重 Panic 就会立刻结束返回 Panic 信息，如果没有发生双重 Panic 就会继续后续处理。

接下来就会运行 Panic 钩子函数，这个钩子函数类似 Global allocator，可以由开发者指定的。不指定的话就会运行默认钩子函数，打印信息到标准错误输出并生成一个 Backtrace。
```rust
pub fn set_hook(hook: Box<dyn Fn(&PanicInfo<'_>) + Sync + Send + 'static>)
```

再之后会运行 Panic 运行时，Panic 运行时有两类，一类是 Abort 运行时，设置了的话这时候就直接结束了。而常规情况下我们会默认设置 Unwind 运行时，这个时候程序就会向上一层一层的展开函数调用栈，并在此运行时中会根据现场情况调用析构函数。

那么问题来了，Unwind 何时终止呢？在`main()`函数运行之前，Rust 运行时会创建线程，将`main()` 函数作为闭包传给`catch_unwind` 函数，再将 `catch_unwind` 作为函数指针去创建线程（这部分内容在编译器内，不容易看到）。

```rust

fn lang_start_internal(...) -> Result<isize, !> {

...
	// catch init 
	// catch main
	let ret_code = panic::catch_unwind(move || panic::catch_unwind(main).unwrap_or(101) as isize)
	    .map_err(move |e| {
	        mem::forget(e);
	        rtabort!("drop of the panic payload panicked");
	    });
	// catch exit
...

}
```

Unwind 过程就是在 `catch_unwind` 这里终止的，Backtrace 会展开到这里。在最后我们会介绍`catch_unwind` 函数和使用方法。

## 标准库 Vec 案例

标准库中的 Vec 这个抽象读者肯定都已经非常熟悉了，而它也能够保证 Panic safety。我们来看 Vec 中 `extend_with(n, value)` 这个函数，他能够让 Vec 拓宽 n 个 T 类型的内存空间，并在这段空间内逐次通过 T 的 `clone()` 方法填充`value: T`。

然而，这里的`clone()`方法就是可能发生 Panic 的地方，毕竟 Clone 是个 User-defined 函数，用户可以写出任意代码，其中就有可能包含 Panic 代码。

这里我们先忽略掉`SetLenOnDrop` 这个用于保证 Panic safety 的临时变量，后续会讲。

首先，Vec 已经通过 `reserve()` 函数将自身扩容成功，然后依次`clone()` value，填充到对应位置上。我们考虑两种情况，一是填充T后将  len 置为扩充后的真实长度，二是填充 T 前将  len 置为扩充后的真实长度。
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

其实我们最希望的是，Vec 在 extend_with 中发生 Panic 时，他的 len 是真实被填充了T的 Vec 长度，可以通过 Vec 访问到这些已经初始化的内存，不会访问到那些未初始化的内存。在 Vec 中这里的 Invariant是指 len 包含的内存长度，都存在有效的T。官方 STD 标准库里面就用 RAII 的机制以一种很 trick 的方法实现了这个能力。

## 奇技淫巧

这个 `SetLenOnDrop<'a>` 就是我们要讨论的奇技淫巧，他能够捕获 Panic，并在 Unwind 过程中将 Vec len 设置为正确的值。

> 注意! Unwind 过程中会调用 Drop，析构掉 Panic 发生之前的局部变量

这里`SetLenOnDrop<'a>`是个局部变量，他包含的 len 是指向 Vec len 的一个可变引用。由于它是在可能发生 Panic 的`clone()` 调用之前被创建的，当发生了 Panic ，Unwind 会尝试去析构这个局部变量，也就会调用到 `SetLenOnDrop<'a> drop()`方法，这个时候 Unwind 会将 Vec 设置为正确的len。

```rust
pub(super) struct SetLenOnDrop<'a> {
    len: &'a mut usize,
    local_len: usize,
}

impl Drop for SetLenOnDrop<'_> {
    #[inline]
    fn drop(&mut self) {
        *self.len = self.local_len;
    }
}
```

这个技巧非常有意思，由于我们知道 Unwind 会析构局部变量，所以翻转一下，我们通过局部变量来捕获 Panic，将我们想要在发生 Panic 时执行的代码逻辑写在 Drop 里，让 Panic 替我们执行一段我们希望的程序逻辑。

## 拓展阅读

### Once Finish

`Once<T>` 在调用 call_once 函数初始化泛型 T 时会调用用户传入的初始化闭包 `f()`，显然这个初始化闭包是用户定义的，用户实现这个函数时可能会发生 Panic。这里通过 `Finish` 这个结构体作为局部变量捕获 `f()`可能发生的 Panic，当 Panic 发生时会类似上述 Vec 一样，在 Unwind 过程中调用 `drop()` 方法，并将`Once<T>`的状态值置为 `Status::Panicked`，这时候如果有其他线程访问 `Once<T>`的时候能够意识它处在一个错误的状态，可以一同 Panic。

显然，我们希望这段 Panic 时执行的 `Finish Drop` 代码逻辑不在正常的代码执行流中，我们可以在函数正常结束时对这个局部变量调用`mem::forget()`方法，避开这段在局部变量 Drop 中执行的代码逻辑。但`mem::forget()` 了之后会不会导致这个`Finish`局部变量内存被泄漏呢？答案是不会的。因为这个局部变量是在栈上分配的，当`try_call_once_slow()` 这个函数返回时，他的栈帧（stack frame）会被释放，rsp 寄存器上移，即使不调用 Drop 方法，也会隐性地释放掉局部变量这块内存空间。

```rust
// Once<T> call_once inner method
fn try_call_once_slow<F: FnOnce() -> Result<T, E>, E>(&self, f: F) -> Result<&T, E> {
	// status transition
	...
	
	// We use a guard (Finish) to catch panics caused by builder
	let finish = Finish {
		status: &self.status,
	};
	// f() is user-defined initialization function
	let val = match f() {
		Ok(val) => val,
		Err(err) => {
			// If an error occurs, clean up everything and leave.
			core::mem::forget(finish);
			self.status.store(Status::Incomplete, Ordering::Release);
			return Err(err);
		}
	};
	unsafe {
		// SAFETY:
		// `UnsafeCell`/deref: currently the only accessor, mutably
		// and immutably by cas exclusion.
		// `write`: pointer comes from `MaybeUninit`.
		(*self.data.get()).as_mut_ptr().write(val);
	};
	core::mem::forget(finish);

	self.status.store(Status::Complete, Ordering::Release);

	// This next line is mainly an optimization.
	return unsafe { Ok(self.force_get()) };
	}
}

struct Finish<'a> {
    status: &'a AtomicStatus,
}

impl<'a> Drop for Finish<'a> {
    fn drop(&mut self) {
        self.status.store(Status::Panicked, Ordering::SeqCst);
    }
}

```
### Mutex Poison

在标准库里面的std::sync::mutex，当你对它上锁的时候，会返回一个带有`MutexGuard<'_, T>`的 `Result`，大多数情况下我们都会忽略它，直接 unwrap()。但其实这样是不正确的。这个 `Result` 就是用来告诉我们，有线程 Panic 导致这个 mutex 中毒了（Poisoned）。

```rust
pub struct Mutex<T: ?Sized> {
    inner: sys::Mutex,
    poison: poison::Flag,
    data: UnsafeCell<T>,
}

impl<T: ?Sized> Mutex<T> {
	pub fn lock(&self) -> LockResult<MutexGuard<'_, T>> {
		unsafe {
			self.inner.lock();
			MutexGuard::new(self)
		}
	}
}

----------

pub struct MutexGuard<'a, T: ?Sized + 'a> {
    lock: &'a Mutex<T>,
    poison: poison::Guard,
}

impl<T: ?Sized> Drop for MutexGuard<'_, T> {
    #[inline]
    fn drop(&mut self) {
        unsafe {
            self.lock.poison.done(&self.poison);
            self.lock.inner.unlock();
        }
    }
}

----------

pub struct Flag {
    failed: AtomicBool,
}

impl Flag {
    pub const fn new() -> Flag {
        Flag {
            failed: AtomicBool::new(false),
        }
    }

    pub fn done(&self, guard: &Guard) {
        if !guard.panicking && thread::panicking() {
            self.failed.store(true, Ordering::Relaxed);
        }
    }
}

--------------

pub struct Guard {
    panicking: bool,
}
```

当用户拿着 mutex 的锁的时候，会持有一个 `MutexGuard` 直到结束临界区的操作，如果用户在临界区发生了 Panic，unwind 过程中会析构这个 `MutexGuard`，在这个过程中会调用 `poison::Flag` 的`done()`方法，如果这个线程确实 Panic了，会给 `Mutex`置上标识位，表示已经Poisoned 了，此时其他线程就能够在lock过程中通过标识位感知到 Panic的发生。

## `catch_unwind` method &  `UnwindSafe` trait 

我们之前提到过，Rust 运行时会通过 `catch_unwind`方法中止 `main` 函数发生 Panic 后的 Unwind 过程。其实开发者也能够在自己的程序中使用 `catch_unwind`方法，用来捕获传入的闭包中可能发生的 Panic，并将 Panic 转化为 Result 然后再进行随后的处理。一般情况下，我们会在 C 或者其他语言调用 Rust 时插入一个`catch_unwind`，毕竟 Unwind 不能展开到 C 语言里面。另外根据安全原则，我们也应该在在一个语言内处理这个语言可能会发生的问题，不要扩散到另一个语言，因此在语言边界上需要小心处理 Panic。

```rust
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R>
```

我们可以看到这个`catch_unwind`方法有个限制，要求这个闭包符合 `UnwindSafe` 类型。`UnwindSafe` trait 其实是类似 `Send` 和 `Sync` 的 marker trait，通过标记告诉编译器这个trait 符合  Panic safety 的要求。

```rust 
pub auto trait UnwindSafe { }
```

对于大部分结构体，只要它能够保证在使用了catch_unwind 之后，很难再观测到一个破坏的 invariant，就是`UnwindSafe`的。而比如`&mut T` 和 `&RefCell<T>` 就不满足`UnwindSafe`。因为可以在 catch_unwind 闭包里面，通过这个可变引用修改这 `T`，但在catch_unwind 结束之后还能够访问到这个`T`，也就是可以被观测到 broken invariant。

而正常情况下，如果传入的是`&T` 和 `T`所有权，无法在闭包里面修改 T，或者修改了能保证他的生命周期在闭包内结束，这两种情况都能够保证即使发生了 panic，broken invariant 也不会穿越 catch_unwind 被访问到。

```rust
let mut x: Vec<i32> = vec![1];
let mut y: Vec<i32> = vec![2];

// 传入&Vec，编译运行成功
panic::catch_unwind( move || {
	x.len();
	panic!("Panic start");
	y.len();
}).ok();

// 传入所有权，编译运行成功
panic::catch_unwind( move || {
	x.push(100);
	panic!("Panic start");
	y.push(100);
}).ok();

// 传入 &mut Vec，编译错误：
// the type `&mut Vec<i32>` may not be safely transferred across an unwind boundary
panic::catch_unwind(|| {
	x.push(100);
	panic!("Panic start");
	y.push(100);
}).ok();

// 传入 &mut Vec，但是通过实现了UnwindSafe trait的 AssertUnwindSafe<T> 作为 wrapper,
// 通过开发者保证unwind safe, 绕开了编译器检查
panic::catch_unwind( AssertUnwindSafe( || {
	x.push(100);
	panic!("Panic start");
	y.push(100);
})).ok();

```

## 总结

Panic safety 是个很强的性质，大部分情况下我们不太需要关注。正常的情况下，我们只需要使用 Result就足够了，只有极端情况下无法处理程序状态时我们才用 Panic 宏。上述的方式提供了一种捕获Panic的方法，并能够在 Panic 发生的时候完成一些事情，保证某个抽象被使用时看起来是 Invariant。

### 参考：
- https://rust-unofficial.github.io/too-many-lists/sixth-panics.html
- https://doc.rust-lang.org/nomicon/exception-safety.html
- https://doc.rust-lang.org/std/panic/trait.UnwindSafe.html
- https://doc.rust-lang.org/std/panic/fn.catch_unwind.html
- https://doc.rust-lang.org/std/panic/fn.set_hook.html
- https://doc.rust-lang.org/std/panic/struct.AssertUnwindSafe.html#
- https://github.com/rust-lang/rfcs/blob/master/text/1236-stabilize-catch-panic.md （catch_unwind RFC）

### 拓展阅读：
- https://doc.rust-lang.org/1.80.1/src/alloc/vec/mod.rs.html#2686 (Vec中的SetLenOnDrop)
- https://docs.rs/spin/latest/src/spin/once.rs.html#252 (Once 中的Finish)
- https://doc.rust-lang.org/src/std/sync/mutex.rs.html#550 (Mutex 中的 MutexGuard poison)
