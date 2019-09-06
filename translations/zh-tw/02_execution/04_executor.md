# 案例：打造一個執行器

Rust 的 `Future` 具有惰性：除非積極驅動到完成，不然不會做任何事。驅使 future 完成的方法之一是在 `async` 函式中用 `.await` 等待它，但這只會將問題推向更高的層次：誰來跑最上層 `async` 函式返回的 future？答案揭曉，我們需要一個 `Future` 執行器（executor）。

`Future` 執行器會拿一群最上層的 `Future` 並在它們可有所進展時呼叫 `poll` 執行它們一直到完成。通常一個執行器會 `poll` 一個 future 一次作為開始。當 `Future` 透過呼叫 `wake()` 表示它們已就緒並可以有所進展時，`Future` 會被放回佇列並再度被呼叫 `poll`，不斷重複直到該 `Future` 完成了。

在這一部分，我們將編寫自己的簡易執行器，能夠並行執行一大群最上層的 future 直到完成。

這個示例中，我們會依賴 `futures` crate 的 `ArcWake` trait，這個 trait 提供一個簡便構建 `Waker` 的方法。

```toml
[package]
name = "xyz"
version = "0.1.0"
authors = ["XYZ Author"]
edition = "2018"

[dependencies]
futures-preview = "=0.3.0-alpha.17"
```

接下來，我們需要在 `src/main.rs` 加上一些 import。

```rust
{{#include ../../examples/02_04_executor/src/lib.rs:imports}}
```

我們的執行器工作模式是發送任務到通道上（channel）。這個執行器會從通道拉取事件並執行。當一個任務已就緒將執行更多工作（被喚醒），它就可將自己放回通道中，以此方式自我排程來等待再次被輪詢。

在這個設計下，執行器僅需要任務通道的接收端。使用者則會取得發送端，並藉由發送端產生（spawn）新的 future。而任務本身實際上是可自我排程的 future，所以我們將任務自身的結構設計為一個 future，加上一個可讓任務重新加入佇列的通道發送端。

```rust
{{#include ../../examples/02_04_executor/src/lib.rs:executor_decl}}
```

我們也在產生器（spawner）新增一個方法，使其產生新 future 更便利。這個方法需要傳入一個 future 型別，會將它 box 起來再放入新建立的 `Arc<Task>` 中，以便於將 future 加入 executor 的佇列。

```rust
{{#include ../../examples/02_04_executor/src/lib.rs:spawn_fn}}
```

欲輪詢 future，我們需要建立 `Waker`。在[任務喚醒一節][task wakeups section]中討論過，讓任務 `wake` 被呼叫時的輪詢排程是 `Waker` 的職責所在。記住這一點，`Waker` 會告知執行器確切已就緒的任務為何，並允許執行器輪詢這些已就緒可有所進展的 future。建立一個新 `Waker` 最簡單的作法是透過實作 `ArcWake` trait，並使用 `waker_ref` 或 `.into_waker()` 函式將 `Arc<impl ArcWake>` 型別轉換為 `Waker`。現在我們來替任務實作 `ArcWake`，允許它們可以轉換成 `Waker` 並可以被喚醒：

```rust
{{#include ../../examples/02_04_executor/src/lib.rs:arcwake_for_task}}
```

當 `Waker` 從一個 `Arc<Task>` 建立後，呼叫它的 `wake()` 會發送一個 `Arc` 副本到任務通道上。我們的執行器就會取出並輪詢該任務。實作如下：

```rust
{{#include ../../examples/02_04_executor/src/lib.rs:executor_run}}
```

恭喜啊！我們現在有一個能動的 future 執行器。我們甚至可以用它來執行
`async/.await` 程式碼或是親手打造的 future，像是之前寫的 `TimerFuture`。

```rust
{{#include ../../examples/02_04_executor/src/lib.rs:main}}
```

[task wakeups section]: ./03_wakeups.md
