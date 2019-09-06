# `Future` Trait

`Future` trait 在 Rust 非同步程式設計中扮演關鍵角色。一個 `Future` 就是一個非同步的運算，產出一個結果值（雖然這個值可能為空，例如 `()`）。一個*簡化*版的 future trait 看起來如下：

```rust
{{#include ../../examples/02_02_future_trait/src/lib.rs:simple_future}}
```

Future 可以透過呼叫 `poll` 函式往前推進，這個函式會盡可能地驅使 future 邁向完成。若 future 完成了，就回傳 `Poll::Ready(reuslt)`。若這個 future 尚無法完成，就回傳 `Poll::Pending`，並安排在該 `Future` 就緒且能有所進展時來呼叫 `wake()` 函式。當呼叫了 `wake()`，執行器（executor）會驅使 `Future` 再次呼叫 `poll`，於是 `Future` 就可以取得更多進展。

若沒有 `wake()`，執行器不會知道哪個 future 能有所進展，因此必須不斷輪詢所有 future。有了 `wake()`，執行器能夠確切知道哪個 future 已準備好接受輪詢。

舉例來說，試想我們要從 socket 讀取一些可能尚未就緒的資料。如果有資料進來，我們則可以讀取它並回傳 `Poll:Ready(data)`；但若沒有任何資料就緒，我們的 future 會被阻塞且無法取得任何進展。當沒有任何資料，我們必須註冊 `wake` 函式，並在 socket 上的資料就緒時呼叫它，進而告知執行器我們的 future 準備好取得進展了。一個簡單的 `SocketRead` future 大致上類似這樣：

```rust
{{#include ../../examples/02_02_future_trait/src/lib.rs:socket_read}}
```

這個 `Future` 原型可以在不需要中間配置（intermediate allocation）下組合多個非同步操作。並可以經由免配置狀態機（allocation-free state machine）實作同時執行多個 future 或是串聯多個 future 的操作，示例如下：

```rust
{{#include ../../examples/02_02_future_trait/src/lib.rs:join}}
```

這展示了多個 future 如何在無需分別配置資源的情形下同時執行，讓我們能夠寫出更高效的非同步程式。無獨有偶，多個循序的 future 也能一個接著一個執行，如下所示：

```rust
{{#include ../../examples/02_02_future_trait/src/lib.rs:and_then}}
```

這些示例展現出 `Future` 可以在無需額外配置的物件與深度巢狀回呼 （nested callback）的情形下，清晰表達出非同步的流程控制。看完這樣基本的流程控制，讓我們來討論真正的 `Future` trait 和剛剛的原型哪裡不相同。

```rust
{{#include ../../examples/02_02_future_trait/src/lib.rs:real_future}}
```

你會發現的第一個改變是 `self` 型別不再是 `&mut self`，而是由 `Pin<&mut self>` 取代。我們會在[往後的章節][pinning]有更多關於 Pinning 討論，此刻知道它允許我們建立一個不移動（immovable）的 future。不移動物件可在它們的欄位（field）保存指標，例如 `struct MyFut { a: i32, ptr_to_a: *const i32 }`。Pinning 是啟用 async/await 前必要之功能。

第二，`wake: fn()` 改為 `&mut Context<_>`。在 `SimpleFuture` 我們透過呼叫一個函式指標（`fn()`）來告知 future 執行器這個 future 可以接受輪詢。然而，由於 `fn()` 是 zero-sized，它不能儲存任何有關*哪個* `Future` 呼叫了 `wake` 的資訊。

在真實世界場景下，一個複雜如網頁伺服器的應用程式可能有數以千計不同的連線，這些連線的喚醒函式需要妥善分別管理。`Context` 型別提供取得一個叫 `Waker` 的型別來解決此問題，`Waker` 的功能則是用來喚醒一個特定任務。

[pinning]: ../04_pinning/01_chapter.md
