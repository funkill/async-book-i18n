> [task-model/finish.md](https://github.com/aturon/apr/blob/ffb00140a767d6e7a4a8875bf6965d10f830a271/src/task-model/finish.md)
> commit ffb00140a767d6e7a4a8875bf6965d10f830a271

# 整合任务执行器与事件循环

至此, 我们已经做了一个简单的执行器, 以在单线程中执行多个任务, 以及一个简单的
事件循环以分发定时事件, 当然用的也是单线程. 现在, 我们将他们整合到一起来构建一个
app, 这个app能够支持任意数量的周期任务"dinging", 而只需要两个OS线程.

为了达成这个目标, 我们需要创建一个`Periodic`任务:

```rust,no_run
struct Periodic {
    // a name for this task
    id: u64,

    // how often to "ding"
    period: Duration,

    // when the next "ding" is scheduled
    next: Instant,

    // a handle back to the timer event loop
    timer: Timer,
}

impl Periodic {
    fn new(id: u64, period: Duration, timer: Timer) -> Periodic {
        Periodic {
            id, period, timer, next: Instant::now() + period
        }
    }
}
```

`period`字段告诉我们要多频繁地打印"ding"信息. 这个实现很直接: 告诉任务是要永远
执行, 并且持续地在每经过一个`period`时间后打印一次信息:

```rust
impl Task for Periodic {
    fn poll(&mut self, wake: &Waker) -> Async<()> {
        // are we ready to ding yet?
        let now = Instant::now();
        if now >= self.next {
            self.next = now + self.period;
            println!("Task {} - ding", self.id);
        }

        // make sure we're registered to wake up at the next expected `ding`
        self.timer.register(self.next, wake);
        Async::Pending
    }
}
```

然后, 把以上的东西都放到一起:

```rust,no_run
fn main() {
    let timer = ToyTimer::new();
    let exec = ToyExec::new();

    for i in 1..10 {
        exec.spawn(Periodic::new(i, Duration::from_millis(i * 500), timer.clone()));
    }

    exec.run()
}
```

这个程序最后产生类似这样的输出:

```
Task 1 - ding
Task 2 - ding
Task 1 - ding
Task 3 - ding
Task 1 - ding
Task 4 - ding
Task 2 - ding
Task 1 - ding
Task 5 - ding
Task 1 - ding
Task 6 - ding
Task 2 - ding
Task 3 - ding
Task 1 - ding
Task 7 - ding
Task 1 - ding
...
```

回过头看, 我们所做的有点魔法: `Periodic`的`Task`实现直接说明*单个任务*所应具有的
行为, 但之后我们把一堆任务交织(interleave)到仅仅两个OS线程! 这就是异步的力量!

## 练习: 多任务同时注册

例子的定时器事件循环有个panic代码: "无法在同一时刻注册任务".

- 可以在实例的程序中遇到这个panic吗?
- 如果我们仅仅是移除了panic, 会发生什么?
- 如何改进代码来完全避免这个问题呢?

## 练习: 逐渐结束程序

`Periodic`和`ToyExec`都被设计成不会停止的.

- 修改`Periodic`, 使得每个实例能够被设置为只会ding固定次数, 然后对应的任务停止.
- 修改`ToyExec`, 使得当没有任务存在的时候, 他会停止执行.
- 测试你的方案!
