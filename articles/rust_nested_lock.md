# Rustc 编译器行为导致了嵌套锁！
## 问题背景
最近在实现 Occlum - Signal syscall 的时候，发现Occlum 会在Exitgroup的过程中 hang 住。经过我们（@段然）对于代码和MIR的一系列分析，发现根本原因在于编译器的行为，编译器隐性释放临时变量的行为和开发者对于编译行为的理解存在差异。笔者发现，即使是 Rust 资深开发者，也可能会在编程过程中忽略掉这些细节，因此，特地写了一篇文章分析了几个典型的案例。

本文旨在从 Rust MIR 层面和读者讲清楚Rustc编译器在何时释放临时变量，以避免在开发过程中写出死锁和嵌套锁的代码逻辑。
## 基于RAII 设计的互斥锁
这一小节先给新人开发者讲述一下在 Rust 当中锁的实现和使用。熟悉内容的开发者可以跳过此节。
```rust
pub struct Mutex<T> {
	val: UnsafeCell<T>,
	// Synchronous variable
	lock: AtomicBool,
}

impl <T> for Mutex<T> {
	/// Acquire the mutex.
	pub fn lock(&self) -> MutexGuard<T> {
		...
	}
	
	/// Release the mutex
	fn unlock(&self) {
		...
	}
}

pub struct MutexGuard<'a, T> {
	mutex: &'a Mutex<T>,
}

impl<'a, T> Deref for MutexGuard<'a, T> {
	type Target = T;
	fn deref(&self) -> &Self::Target {
		unsafe { &*self.mutex.val.get() }
	}
}

impl<'a, T> DerefMut for MutexGuard<'a, T> {
	fn deref_mut(&mut self) -> &mut Self::Target {
		unsafe { &mut *self.mutex.val.get() }
	}
}

impl<'a, T> Drop for MutexGuard<'a, T> {
	fn drop(&mut self) {
		self.mutex.unlock();
	}
}
```
这是一个简化版的互斥锁的实现，当开发者想要进入临界区，读写访问被保护的数据变量 `T` 时，需要先调用 `lock()` 方法，在满足同步条件的情况下，**持有互斥锁**，并获取一个互斥锁的 `Guard` 。并通过 `Guard` 读写访问真实数据。而当 `Guard` 被释放时（生命周期结束），会通过 `Guard` 的 `Drop` 方法**隐性地释放互斥锁**。这是一种RAII的设计模式，锁的持有周期等于 `Guard` 结构体的生命周期。  

为了便于后续理解，这里加入了 `MutexGuard` 中 `Deref` 和 `DerefMut` 这两个 `trait` 的实现。这两个 `trait` 让 `MutexGuard` 成为了实至名归的 `T` , 它其实是一个智能指针，能够让开发者在访问 `MutexGuard` 的时候能自动解引用来访问 `T`。  

而何时调用 `Drop trait` 的 `drop()` 方法呢？可以是开发者**显性地**调用 `drop(guard);` 这个表达式，亦或是在 `MutexGuard<'a, T>` 变量的生命周期结束时，编译器回收（释放） `MutexGuard<'a, T>` 的资源前，**隐性地**调用 `drop()` 方法。  

众所周知，Rust 能够保证局部变量会在 `{...}` 花括号结束之前释放掉，但释放顺序、何时释放，是由编译器决定的。Rust 作为一门高级语言，拥有着生命周期结束时回收结构体资源等特性，Rustc 编译器为开发者做的幕后工作减轻了心智负担。但如果开发者在不了解 Rustc 编译器行为的情况下，使用了编译器隐性的高级特性时，一些 `Undefined behaviour` 就发生了，比如嵌套锁！

## 编译器何时释放临时变量！
此节主要以笔者此时最新的Rustc编译器版本为例，适用于 Rustc 1.74.1 stable channel。
> 插曲：[Rust Playground](https://play.rust-lang.org/)（ Rust 在线试用平台）是个非常便捷的工具，能够让开发者快速地做一些小实验，同时也能通过 MIR 和 ASM 看到Rustc的编译的中间结果。
### 前置条件
```rust
use spin::Mutex;

let queue1 = [1; 5].to_vec();
let queue2 = [2; 5].to_vec();

let lock1 = Mutex::new(queue1);
let lock2 = Mutex::new(queue2);
```
### 短路逻辑
```rust
let empty = {
	...
	//  Guard1 -> Vec               Guard2 -> Vec
	lock1.lock().is_empty() || lock2.lock().is_empty()
};
```
#### 太长不看版：
逻辑符号拆分得到了两个表达式语句，编译器会在每个语句执行完后释放用到的临时变量，被逻辑符号区分开的两个语句中上**持有锁的生命周期并不重叠**，不会嵌套锁。

> **注意**：在老版本2022-10-31 中的 Rustc的编译行为和最新Rustc的编译行为并不相同！  
在老版本的编译器中，会持有中间产生的临时变量直到**整个**表达式执行完毕。意味着第一个条件语句执行完之后，会持有lock1 执行第二个条件语句，会导致**两把锁的生命周期相互重叠**，产生嵌套锁。
### 执行流：

上述逻辑是，需要通过拿锁判断两个 queue 是否为空，如果有一个 queue 为空则返回 true 。  

在最新版本的Rustc 编译器中，笔者通过分析MIR 代码，这段代码的编译后的代码执行流如下：

1. 先给互斥锁 lock1 上锁，然后返回一个 guard1 的结构体 (中间变量 / 临时变量)
2. 通过 guard1 的 Deref 解引用能力，直接调用 Vec 的 `is_empty()` 方法
3. 如果返回 true -> drop(guard1) -> 释放 lock1 锁 -> 给 `empty` 变量赋值 true
4. 如果返回 false  -> drop(guard1) -> 释放 lock1 锁 -> 执行第二条语句
5. 给 lock2 上锁 -> 返回 guard2  -> ... 执行逻辑同 2 -> ( 3 / 4 )

MIR 如下：
```
  bb3: {
      _23 = const false;
      _8 = move _3;
      _7 = spin::mutex::Mutex::<Vec<i32>>::new(move _8) -> [return: bb4, unwind: bb21];
  }

  // _14 是lock1的MutexGuard, 持有lock1
  bb4: {
      _15 = &_5;
      _14 = spin::mutex::Mutex::<Vec<i32>>::lock(move _15) -> [return: bb5, unwind: bb20];
  }

  // lock1 MutexGuard 解引用
  bb5: {
      _13 = &_14;
      _12 = <spin::MutexGuard<'_, Vec<i32>> as Deref>::deref(move _13) -> [return: bb6, unwind: bb19];
  }

  bb6: {
      _11 = _12;
      _10 = Vec::<i32>::is_empty(move _11) -> [return: bb7, unwind: bb19];
  }

  // 第一条语句执行结束，判断返回给empty赋值 或者 执行第二条语句
  bb7: { 
      switchInt(move _10) -> [0: bb10, otherwise: bb8];
  }

  // 释放lock1锁
  bb8: {
      drop(_14) -> [return: bb9, unwind: bb20];
  }

  bb9: {
      _9 = const true;
      goto -> bb15;
  }

  // 释放lock1锁
  bb10: {
      drop(_14) -> [return: bb11, unwind: bb20];
  }

  // 执行第二条语句，_19 是lock2的MutexGuard，持有lock2
  bb11: {
      _20 = &_7;
      _19 = spin::mutex::Mutex::<Vec<i32>>::lock(move _20) -> [return: bb12, unwind: bb20];
  }

  bb12: {
      _18 = &_19;
      _17 = <spin::MutexGuard<'_, Vec<i32>> as Deref>::deref(move _18) -> [return: bb13, unwind: bb18];
  }

  bb13: {
      _16 = _17;
      _9 = Vec::<i32>::is_empty(move _16) -> [return: bb14, unwind: bb18];
  }

  // 释放lock2锁
  bb14: {
      drop(_19) -> [return: bb15, unwind: bb20];
  }

  bb15: {
      drop(_7) -> [return: bb16, unwind: bb21];
  }

  bb16: {
      drop(_5) -> [return: bb17, unwind: bb24];
  }

  bb17: {
      _23 = const false;
      _24 = const false;
      return;
  }
```

老版本Rustc版本（2022-10-31）执行流：  
在上述的短路逻辑中，需要同时获取两把互斥锁，判断两个queue里面是否有一个为空。在老版本 Rustc 编译器中，这段代码的编译后的代码执行流是：

1. 先给互斥锁 **lock1 上锁**，然后返回一个 guard1 的结构体 (中间变量 / 临时变量)
2. 通过 guard1 的 Deref 解引用能力，直接调用 Vec 的 `is_empty()` 方法
3. 如果返回 true -> **drop(guard1) -> 释放lock1锁** -> 给 `empty` 变量赋值 true （**结束**）
4. 如果返回false  -> 执行第二条语句
5. 给 lock2 上锁 -> 返回 guard2
6. 通过 guard2 的 Deref 解引用能力，直接调用Vec的`is_empty()`方法
7. 无论返回true 或false
8. **drop(guard2) -> 释放lock2锁 -> drop(guard1) -> 释放lock1锁**
9. 将 guard2.is_empty() 结果赋值 `empty`变量（**结束**）

显然，在老版本Rustc编译器中，即使是在短路逻辑的表达式中，中间变量（局部变量）的生命周期也会持续到整个表达式结束。因此，在老版本的Rustc中，如果短路逻辑不同语句中会持有多把锁，建议将短路逻辑表达式拆成 If 语句避免锁嵌套。

#### 表达式
```rust
let val = lock1.lock().pop().or_else(||
	lock2.lock().pop()
);
```

这里举了一个例子，试图给 lock1 上锁，然后从 queue1 中弹出一个数值，如果 queue1 为空，即返回值为 None，则试图给 lock2 上锁，然后从 queue2 中弹出一个数值。那么问题来了，如果lock1 为空，对lock1的持有锁Guard 会持续到 queue2 弹出数值么，会和 lock2 Guard 有重叠的生命周期么？答案是会的，**lock1 Guard会在整个表达式完成时释放！**。

在MIR中，这段代码的执行流是：

1. 先给互斥锁 **lock1 上锁**，然后返回一个guard1的结构体 (中间变量 / 临时变量)
2. 通过guard1的 Deref 解引用能力，直接调用Vec的`pop()`方法
3. 如果返回Some(u32) -> **drop(guard1) -> 释放lock1锁** -> 给`val`变量赋值 Some(u32) （**结束**）
4. 如果返回None  -> 执行闭包 -> 给 lock2上锁 -> 返回guard2
5. 通过 guard2的 Deref 解引用能力，直接调用Vec的`pop()`方法
6. **drop(guard2) -> 释放lock2锁 -> 闭包完成 -> drop(guard1) -> 释放lock1锁**
7. 将 `pop()`结果赋值 `val`变量（**结束**）

可以看到，当表达式中获取锁 Guard，并在获取锁之后执行闭包，Guard的生命周期会持续到闭包结束，很容易造成嵌套锁。为了避免冗余，此处不写出 MIR ，感兴趣的同学可以在 Playground 上生成。  
客观的说，确实上述代码看起来更加简洁，但避免程序故障比代码优雅的优先级更高，建议修改上述代码如下：
```rust
let mut val = lock1.lock().pop();
if val.is_none() {
	val = lock2.lock().pop();
}
```
## Code Style Tips
### 时刻关注锁的生命周期
#### Case 1
在写代码时，应当时刻关注Guard (持有锁)的生命周期，如果拿不准编译器的行为，尽量主动调用 `drop()` 方法。或者用 `{}` 花括号收束 Guard 生命周期。如：
```rust
let guard = lock1.lock();
// Touch lock inner data
...
drop(guard)；

// Do something else
...

--------------------------

let retval = {
	let guard = lock1.lock();
	// Touch lock inner data
	...
	retval
};
```
#### Case 2
当拿到 Guard 之后，尽量避免长时间持有 Guard 引用。以下给了两个持有 Guard 的例子，不建议按照例子A的方式编程，因为会在`handle()`这个方法执行期间持有锁。例子中给出的 `handle()` 方法十分简单，但如果包含复杂逻辑，如获取锁等操作，就会非常容易陷入死锁难题。而以例子B的方式，互斥锁保护的数据 u32 会以 Copy 的方式 move 到 `val` 变量上，互斥锁会在取完内部数据之后释放掉，在执行 `handle()` 期间并不会持有lock锁。
```rust
fn handle(val: &u32) {
	println!("val is {:?}", val);
}

let lock = Mutex::new(1_u32);

// Case A: Not Recommended

// Here `guard` type is MutexGuard<u32>
let guard = lock.lock();
// Auto dereference
handle(&guard);
// Guard live longer than handle() method

--------------------------
// Case B: Recommended

// Here `val` type is u32, inexplicitly drop(MutexGuard) before copying u32
let val = *lock.lock();
// Parse &u32
handle(&val);
```

#### Case 3
这个例子其实和 case2 是比较接近的。以本例 Case A 的写法，即使在最新的 Rustc 编译器中，仍然会产生嵌套锁。这个例子会先给 a_lock 上锁，拿到 a_lock 的 a_guard ，解引用 a_lock 拿到a的值，然后对 b_lock 上锁，拿到 b_lock 的 b_guard，解引用b_lock拿到b的值，待a和b做完位运算之后，再去释放a_guard 和 b_guard，导致了嵌套锁的问题。  

而以Case B的写法，a_guard 会在给变量a赋值前释放，所以锁Guard的生命周期并不会和另一个上锁周期发生重叠，因此推荐 Case B的写法。
```rust
let a_lock = Mutex::new(1_u32);
let b_lock = Mutex::new(2_u32);

// Case A: Not Recommended
let a_b = *a_lock.lock() | *b_lock.lock(); 

// Case B: Recommended
let a = *a_lock.lock();
let b = *b_lock.lock();
let a_b = a | b;
```
### 不得已嵌套时，锁的顺序很重要
如果在无法通过设计模式消除掉嵌套锁时，需要关注锁的持有顺序，尽量保证在不同执行流的逻辑内，锁的执行顺序保持一致，这样能够规避掉 "线程a 拿锁lock1，再拿锁lock2；线程b 拿锁lock2，再拿锁lock1“ 这种标准的死锁情况。
```rust
let guard1 = lock1.lock();
let guard2 = lock2.lock();
// do something
...
drop(guard1);
drop(guard2);
```
## 最后
希望读完本文之后，衷心希望读者能够在处理恼人的死锁问题时更加游刃有余。
