# 透過 ``Waker`` 喚醒任務

對 future 來說，第一次被輪詢（`poll`）時任務尚未完成再正常不過了。當這件事發生，future 必須確認它未來就緒時會被再度輪詢以取得進展。這就需透過 `Waker` 型別處理。

每次輪詢一個 future 時，都將其作為「任務（task）」的一部分來輪詢。任務實際上是已提交到執行器的最上層 future。

`Waker` 提供一個 `wake()` 方法，可以告知執行器相關的任務需要喚醒。當呼叫了 `wake()`，執行器就會得知連動該 `Waker` 的任務已準備就緒取得進展，且應該再度輪詢它的 future。

`Waker` 同時實作了 `clone()`，所以它可以四處複製與儲存。

我們來試試用 `Waker` 實作一個簡易計時器。

## 案例：打造一個計時器

為了貼近示例的目的，我們只需要在計時器建立時開一個執行緒，睡眠一段必要的時間，當計時器時間到，再發送訊號給計時器 future。

這邊是我們開始手作前需要的 import：

```rust
{{#include ../../examples/02_03_timer/src/lib.rs:imports}}
```

讓我們從定義自己的 future 型別開始。我們的 future 需要一個溝通方法，給執行緒通知我們計時器的時間到了 future 也該完成了。我們會用一個共享狀態 `Arc<Mutex<..>>` 在執行緒與 future 間溝通。

```rust
{{#include ../../examples/02_03_timer/src/lib.rs:timer_decl}}
```

現在，我們要實際動手實作 `Future` 了！

```rust
{{#include ../../examples/02_03_timer/src/lib.rs:future_for_timer}}
```

很簡單吧？如果執行緒設定 `shared_state.completed = true`，我們完成了！否則，我們就複製當前任務的 `Waker` 並將其傳入 `shared_state.waker`，以便執行緒往後喚醒任務。

重要的是，在每次輪詢 future 後必須更新 `Waker`，因為 future 有可能移動到帶有不同 `Waker` 的不同任務上。這種狀況會發生在 future 在 task 之間相互傳遞時。

最後，我們需要可以實際構建計時器與啟動執行緒的 API：

```rust
{{#include ../../examples/02_03_timer/src/lib.rs:timer_new}}
```

呼！這些就是打造簡易計時器 future 的一切所需。現在，我們若有一個執行器可以讓  future 跑起來⋯
