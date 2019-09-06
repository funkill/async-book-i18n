# `select!`

`futures::select` 巨集可以同時執行多個 future，並在任何一個 future 完成是迅速回應使用者。

```rust
{{#include ../../examples/06_03_select/src/lib.rs:example}}
```

上面這個函式將並行執行 `t1` 與 `t2`。當 `t1` 或 `t2` 其中一個完成了，對應的處理程序就會呼叫 `println!`，這個函式就不會完成剩下的任務，而會直接結束。

`select` 基本的語法是 `<pattern> = <expression> => <code>`，你想要從多少個 future 中做 `select` 就重複多少次。

## `default => ...` 與 `complete => ...`

`select` 也支援 `default` 與 `complete` 的流程分支。

`default` 分支會在所有被 `select` 的 future 都尚未完成時。一個包含 `default` 分支的 `select` 總是會立刻返回，因為開始執行時不會有任何 future 準備就緒。

`complete` 分支則負責處理這個狀況：當所有被 `select` 的 future 都完成且無法再有更多進展時。這有個不斷循環 `select` 的小撇步。

```rust
{{#include ../../examples/06_03_select/src/lib.rs:default_and_complete}}
```

## 與 `Unpin` 及 `FusedFuture` 的互動

你可能會發現，第一個例子中我們會對兩個 `async fn` 回傳的 future 呼叫 `.fuse()`，且用了 `pin_mut` 來固定它們。這兩個都是必要操作，因為在 `select` 中的 future 必須實作 `Unpin` trait 與 `FusedFuture` trait。

`Unpin` 必要的原因是用在 `select` 的 future 不是直接取其值，而是取其可變引用（mutable reference）。由於沒有取走所有權，尚未完成的 future 可以在 `select` 之後再度使用。

`FusedFuturue` 列為必須是因為當一個 future 完成時，`select` 就不該再度輪詢它。`FusedFuture` 的原理是追蹤其 future 是否已完成。實作了 `FusedFuture` 使得 `select` 可以用在迴圈中，只輪詢尚未完成的 future。這可以從上面的範例中看到，`a_fut` 或 `b_fut` 將會在第二次迴圈迭代中完成，由於 `future::ready` 有實作 `FusedFuture`，所以可以告知 `select` 不要再輪詢它。

注意，stream 也有對應的 `FusedStream` trait。有實作這個 trait 或使用 `.fuse()` 封裝的 stream，都會從它們的組合子 `.next()` / `.try_next()` 產生 `FusedFuture`。

```rust
{{#include ../../examples/06_03_select/src/lib.rs:fused_stream}}
```

## 在 `select` 迴圈中使用 `Fuse` 與 `FuturesUnordered` 的並行任務

有個難以發現但很方便的函式是 `Fuse::terminated()`，它可以建立一個空的且已經終止的 future，並在往後填上實際需要執行的 future。

`Fuse::termianted()` 在遇到需要在 `select` 迴圈內執行一個任務，但這任務卻在在該迴圈內才產生的情況，顯得十分方便順手。

請注意使用 `.select_next_some()` 函式。這個函式可以與 `select` 配合，只執行會從 steam 產生 `Some(_)` 的分支，而忽略產生 `None` 的分支。

```rust
{{#include ../../examples/06_03_select/src/lib.rs:fuse_terminated}}
```

當同一個 future 的多個副本需要同時執行，請使用 `FuturesUnordered` 型別。接下來的範例和上一個很相似，但會執行每個 `run_on_new_num_fut` 的副本到其完成，而非當新的產生就中止舊的。它同時會印出 `run_on_new_num_fut` 的返回值。

```rust
{{#include ../../examples/06_03_select/src/lib.rs:futures_unordered}}
```
