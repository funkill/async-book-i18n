# 目錄

- [開始上手](01_getting_started/01_chapter.md)
  - [為什麼是非同步？](01_getting_started/02_why_async.md)
  - [非同步 Rust 的現況](01_getting_started/03_state_of_async_rust.md)
  - [`async`/`.await` 入門](01_getting_started/04_async_await_primer.md)
  - [案例：HTTP 伺服器](01_getting_started/05_http_server_example.md)
- [揭秘：執行 `Future` 與任務](02_execution/01_chapter.md)
  - [`Future` Trait](02_execution/02_future.md)
  - [透過 `Waker` 喚醒任務](02_execution/03_wakeups.md)
  - [案例：打造一個執行器](02_execution/04_executor.md)
  - [執行器與系統輸入輸出](02_execution/05_io.md)
- [`async`/`await`](03_async_await/01_chapter.md)
- [Pinning](04_pinning/01_chapter.md)
- [Streams](05_streams/01_chapter.md)
  - [迭代與並行](05_streams/02_iteration_and_concurrency.md)
- [同時執行多個 Future](06_multiple_futures/01_chapter.md)
  - [`join!`](06_multiple_futures/02_join.md)
  - [`select!`](06_multiple_futures/03_select.md)
  - [TODO: 產生 Spawning](404.md)
  - [TODO: 取消與逾時](404.md)
  - [TODO: `FuturesUnordered`](404.md)
- [TODO: I/O](404.md)
  - [TODO: `AsyncRead` 與 `AsyncWrite`](404.md)
- [TODO: 非同步設計模式：解法與建議](404.md)
  - [TODO: 打造伺服器與請求/回應模式](404.md)
  - [TODO: 共享狀態管理](404.md)
- [TODO: 生態系：Tokio 與更多](404.md)
  - [TODO: 還有好多好多？...](404.md)