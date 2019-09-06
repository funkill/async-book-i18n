> [task-model/real/tasks.md](https://github.com/aturon/apr/blob/ffb00140a767d6e7a4a8875bf6965d10f830a271/src/task-model/real/tasks.md)
> commit ffb00140a767d6e7a4a8875bf6965d10f830a271

# 任务

`futures`库没有直接定义`Task`trait, 取而代之地定义了更加通用的概念*futures*,
我们很快将深入了解它的细节. 现在, 让我们看看future的核心定义吧:

```rust,no_run
/// An asynchronous computation that completes with a value or an error.
trait Future {
    type Item;
    type Error;

    /// Attempt to complete the future, yielding `Ok(Async::Pending)`
    /// if the future is blocked waiting for some other event to occur.
    fn poll(&mut self, cx: task::Context) -> Poll<Self::Item, Self::Error>;
}

type Poll<T, E> = Result<Async<T>, E>;
```

Futures很像任务, 除了他们返回的是一个result(这允许它们进行组合). 换言之, 你可以
将*任务*理解为是任意实现了`Future<Item = (), Error = !>`的类型

然而, 它们还有其他区别: Futures不需要`WakeHandle`参数. 实践中, 这个参数几乎总是
从执行器处创建到将句柄(handle)排队时被传递下来并保持不变, 因此, 在`futures`库
里, 执行器在现成局部变量里面提供了方便的`WakeHanle`. 你可以用`futures::task`里
的`curent_wake`函数来获得它:

```rust,no_run
/// When called within a task being executed, returns the wakeup handle for
/// that task. Panics if called outside of task execution.
fn current_wake() -> WakeHandle;
```

## 理解`Future`和`ToyTask`的关联

能够知道`Future`和`ToyTask`如何*精确地*联系在一起是很有帮助的. 为此, 我们引入了
一个包装类型来将`ToyTask`转换成`Future`:

```rust,no_run
use futures::task;
use futures::{Future, AsyncResult};

struct ToyTaskToFuture<T>(T);

impl<T: ToyTask> Future for ToyTaskToFuture<T> {
    type Item = ();
    type Error = !;

    fn get(&mut self) -> AsyncResult<(), !> {
        Ok(self.0.complete(task::current_wake()))
    }
}
```
