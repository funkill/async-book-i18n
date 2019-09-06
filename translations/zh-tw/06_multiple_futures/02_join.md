# `join!`

`futures::join` 巨集能夠在等待多個不同的 future 完成的同時，並行執行這些 future。

當執行多個非同步操作時，很容易就寫出簡單的循序 `.await`：

```rust
{{#include ../../examples/06_02_join/src/lib.rs:naiive}}
```

However, this will be slower than necessary, since it won't start trying to
`get_music` until after `get_book` has completed. In some other languages,
futures are ambiently run to completion, so two operations can be
run concurrently by first calling the each `async fn` to start the futures,
and then awaiting them both:

然而，因為 `get_music` 不會在 `get_book` 完成後自動嘗試開始，這將比我們需要的更慢。在部分語言中，futurue 在哪都能執行到完成，所以兩個操作可透過第一次呼叫 `async fn` 來開始 future，並在之後等待它們兩者。

```rust
{{#include ../../examples/06_02_join/src/lib.rs:other_langs}}
```

不過，Rust 的 future 在直接正面地 `.await` 之前不會做任何事。這代表這兩段程式碼小片段將不會並行執行，而是循序執行 `book_future` 與 `music_future`。想正確並行執行兩個 future，請愛用 `futures::join!`：

```rust
{{#include ../../examples/06_02_join/src/lib.rs:join}}
```

`join!` 的返回值是一個帶有每個傳入的 `Future` 的輸出的元組（tuple）。

The value returned by `join!` is a tuple containing the output of each
`Future` passed in.

## `try_join!`

For futures which return `Result`, consider using `try_join!` rather than
`join!`. Since `join!` only completes once all subfutures have completed,
it'll continue processing other futures even after one of its subfutures
has returned an `Err`.

若 future 回傳的是 `Result`，請考慮使用 `try_join!` 而非 `join!`。由於 `join!` 會在所有 subfuture 完成就完成自己，甚至有 subfuture 回傳 `Err` 時還是會繼續處理其他 future。

和 `join!` 不同的事，當任意一個 subfuture 回傳錯誤時，`try_join!` 就會立刻完成。

```rust
{{#include ../../examples/06_02_join/src/lib.rs:try_join}}
```

請注意，傳入 `try_join!` 的 future 需要有同樣的錯誤型別。可以考慮使用 `futures::future::TryFutureExt` 的 `.map_err(|e| ...)` 和 `err_into()` 函式來統一這些錯誤型別：

```rust
{{#include ../../examples/06_02_join/src/lib.rs:try_join_map_err}}
```
