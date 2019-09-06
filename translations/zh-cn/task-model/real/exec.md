> [task-model/real/exec.md](https://github.com/aturon/apr/blob/01ce17e6d1a62ffc514099e78b279399673ef2f9/src/task-model/real/exec.md) 
> commit 01ce17e6d1a62ffc514099e78b279399673ef2f9

# 执行器

First off, in the `futures` crate, executors are objects that can spawn a
`Future<Item = (), Error = !>` as a new task. There are two key executors to be
familiar with.
首先, 在`futures`库里, 执行器是能够产生作为(spawn)新任务的
`Future<Item = (), Error = !>`对象的对象. 有两种关键的执行器需要熟悉

## 线程池(`ThreadPool`)执行器

The simplest executor is `futures::executor::ThreadPool`, which schedules tasks
onto a fixed pool of OS threads. Splitting the tasks across multiple OS threads
means that even if a particular `tick` invocation takes a long time, other tasks
can continue to make progress on other threads.
最简单的执行器是`futures::executor::ThreadPool`, 能够在固定OS线程数量的线程池上
调度任务. 将任务跨线程分离意味着如果一个特定的`tick`调度花费太长的时间, 其他任务
能够在其他线程上继续进行.

初始化和使用也是很直接:

```rust,no_run
// Set up the thread pool, which spins up worker threads behind the scenes.
let exec = ThreadPool::new();

// Spawn tasks onto the thread pool.
exec.spawn(my_task);
exec.spawn(other_task);
```

后面我们会看到一系列沟通已产生任务的不同方法.

请注意, 因为任务会在任意线程上执行, 所以它需要满足`Send`和`'static`

## 当前线程(`CurrentThread`)执行器

`futures`库也提供了一个单线程执行器`CurrentThread`, 类似我们创建的那个玩具版本.
它和线程池执行器的关键不同点是`CurrentThread`能够执行非`Send`和非`'static`任务,
因为该执行器是通过在当前线程显式地被调用:

```rust,no_run
// start up the CurrentThread executor, which by default runs until all spawned
// tasks are complete:
CurrentThread::run(|_| {
    CurrentThread::spawn(my_task);
    CurrentThread::spawn(other_task);
})
```

`ThreadPool`和`CurrentThread`执行器之间的取舍会在
[本书后面章节](async-in-practice/concurrency.html)中详细解释.

## 伪唤醒(Spurious wakeups)

总的来说, 执行器保证了它们会在任务被唤醒的任何时候去调用对应的`get`方法. 然而,
它们*也*可以在其他时候调用`get`. 因此, 任务不能假设每次`get`的调用都有进展; 它们
应该经常重试之前阻塞了它们的操作, 并且准备再次等待

## 练习

- 用`ThreadPool`执行器重写前面小结的例子
- 用`CurrentThread`执行器重写前面小结的例子
- 对于timer例子, 使用这两种执行器的取舍是什么?
