> [task-model/exec.md](https://github.com/aturon/apr/blob/ffb00140a767d6e7a4a8875bf6965d10f830a271/src/task-model/exec.md)
> commit ffb00140a767d6e7a4a8875bf6965d10f830a271

# 一个玩具任务执行器

来造一个任务执行器吧! 我们的目标是使任意数量的任务能够协调运行在*单个*OS线程上
成为可能. 为了保持示例简单, 我们挑选了最原始的数据结构来实现. 以下是我们需要
导入的数据结构:

```rust
use std::mem;
use std::collections::{HashMap, HashSet};
use std::sync::{Arc, Mutex, MutexGuard};
use std::thread::{self, Thread};
```

首先, 我们定义一个保持执行器状态的结构. 该执行器拥有任务本身, 并分配 给每个任务
一个`usize`ID, 使得我们能够从外部指向这些任务. 特殊的, 执行器保持一个`ready`
集合, 存储应该要被唤醒的任务id(因为这些任务在等待的事件已经发生了):

```rust
// the internal executor state
struct ExecState {
    // The next available task ID.
    next_id: usize,

    // The complete list of tasks, keyed by ID.
    tasks: HashMap<usize, TaskEntry>,

    // The set of IDs for ready-to-run tasks.
    ready: HashSet<usize>,

    // The actual OS thread running the executor.
    thread: Thread,
}
```

执行器本身只是将这个状态用`Arc`和`Mutex`包装起来, 允许它能够被其他线程*使用*
(即使所有任务都可以局部线程中运行):

```rust
#[derive(Clone)]
pub struct ToyExec {
    state: Arc<Mutex<ExecState>>,
}
```

现在, `ExecState`的`tasks`字段提供了`TaskEntry`实例, 这些实例包装了一个具体任务, 
和对应的`Waker`:

```rust
struct TaskEntry {
    task: Box<ToyTask + Send>,
    wake: Waker,
}
```

最后, 我们需要一些创建执行器并修改执行器状态的模板:

```rust
impl ToyExec {
    pub fn new() -> Self {
        ToyExec {
            state: Arc::new(Mutex::new(ExecState {
                next_id: 0,
                tasks: HashMap::new(),
                ready: HashSet::new(),
                thread: thread::current(),
            })),
        }
    }

    // a convenience method for getting our hands on the executor state
    fn state_mut(&self) -> MutexGuard<ExecState> {
        self.state.lock().unwrap()
    }
}
```

有了这些模板代码, 我们可以开始关注于执行器的内部工作原理. 我们最好从核心执行器
循环开始. 简便起见, 这个循环从来不退出; 它的职责仅仅是继续跑完所有分出线程的
新任务:

```rust
impl ToyExec {
    pub fn run(&self) {
        loop {
            // Each time around, we grab the *entire* set of ready-to-run task IDs:
            let mut ready = mem::replace(&mut self.state_mut().ready, HashSet::new());

            // Now try to `complete` each ready task:
            for id in ready.drain() {
                // we take *full ownership* of the task; if it completes, it will
                // be dropped.
                let entry = self.state_mut().tasks.remove(&id);
                if let Some(mut entry) = entry {
                    if let Async::Pending = entry.task.poll(&entry.wake) {
                        // The task hasn't completed, so put it back in the table.
                        self.state_mut().tasks.insert(id, entry);
                    }
                }
            }

            // we'd processed all work we acquired on entry; block until more work
            // is available. If new work became available after our `ready` snapshot,
            // this will b no-op.
            thread::park();
        }
    }
}
```

这里精妙的地方是, 在每一轮循环, 我们*在开始的时候*`poll`所有准备好的东西, 然后
"park"线程. `std`库的[`park`]/[`unpark`]让处理阻塞和唤醒OS线程变得很容易. 在
这个例子里, 我们想要的是执行器底层的OS线程阻塞, 直到一些额外的任务已经就绪. 这样
的话, 即使我们无法唤醒线程, 也不会有任何风险: 如果另外的线程在我们初次读取
`ready`集与调用`park`方法之间调用了`unpark`方法, `park`方法就会马上返回.

[`park`]: https://static.rust-lang.org/doc/master/std/thread/fn.park.html
[`unpark`]: https://static.rust-lang.org/doc/master/std/thread/struct.Thread.html#method.unpark

另一方面, 一个任务会像这样被唤醒:

```rust
impl ExecState {
    fn wake_task(&mut self, id: usize) {
        self.ready.insert(id);

        // *after* inserting in the ready set, ensure the executor OS
        // thread is woken up if it's not already running.
        self.thread.unpark();
    }
}

struct ToyWake {
    // A link back to the executor that owns the task we want to wake up.
    exec: ToyExec,

    // The ID for the task we want to wake up.
    id: usize,

impl Wake for ToyWake {
    fn wake(&self) {
        self.exec.state_mut().wake_task(self.id);
    }
}
```

剩下的部分就很直接了. `spawn`方法负责将任务包装成`TaskEntry`并执行它:

```rust
impl ToyExec {
    pub fn spawn<T>(&self, task: T)
        where T: Task + Send + 'static
    {
        let mut state = self.state_mut();

        let id = state.next_id;
        state.next_id += 1;

        let wake = ToyWake { id, exec: self.clone() };
        let entry = TaskEntry {
            wake: Waker::from(Arc::new(wake)),
            task: Box::new(task)
        };
        state.tasks.insert(id, entry);

        // A newly-added task is considered immediately ready to run,
        // which will cause a subsequent call to `park` to immediately
        // return.
        state.wake_task(id);
    }
}
```

好了, 我们造了个任务调度器了! 但是在高兴之余, 一个很重要的地方是, 我们发现前面的
实现会因为`Arc`循环引用而导致任务泄露. 试着解决这个问题吧, 这是个很好的练习.

但是, 就算解决了上面的问题, 它仍然是个"玩具"执行器. 现在, 让我们继续造个事件源,
让任务等待去处理.

