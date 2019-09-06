# `async`/`.await`

在[第一章][the first chapter]，我們簡要介紹了 `async/.await`，並用它簡單架構一個簡易伺服器。本章節將深入探討 `async/.await` 的細節，解釋其運作原理，並比較 `async` 程式碼和傳統 Rust 程式的區別。

`async`/`.await` 是特殊的 Rust 語法，使其能轉移控制權到當前執行緒，而非阻塞之，並在等待操作完成的同時，允許其他程式碼繼續推進。

使用 `async` 有兩個主要途徑：`async fn` 與 `async` 區塊（block）。兩者皆返回一個實作 `Future` trait 的值：

```rust
{{#include ../../examples/03_01_async_await/src/lib.rs:async_fn_and_block_examples}}
```

如我們在第一章所見， `async` 函式主體和其他 future 都具有惰性：在執行前不做任何事。執行一個 `Future` 最常見的手段是 `.await` 它。當對一個 `Future` 呼叫 `.await` 時，會嘗試執行它到完成。若該 `Future` 阻塞，將會轉移控制權到當前的執行緒。而執行器會在該 `Future` 能取得更多進展時恢復執行它，讓 `.await` 得以解決。

## `async` 生命週期

和其他傳統函式不同， `async fn` 會取得引用（reference）或其他非 `'static` 引數（argument），並返回一個綁定這些引數生命週期的 `Future`。

```rust
{{#include ../../examples/03_01_async_await/src/lib.rs:lifetimes_expanded}}
```

這代表了從 `async fn` 返回的 future 只能在其非 `'-static` 引數的有效生命週期內被 `await`。

在常見的例子像在呼叫函式（如 `foo(&x).await`）立刻 `await` 該 future，這並不構成問題。不過，如果想保存或將這個 future 傳送到其他任務或執行緒中，這可能會發生問題。

常見將一個引用作為引數的 `async fn` 轉換為回傳一個 `'static` future 的方法是，將呼叫 `async fn` 需要的引數包裹在一個 `async` 區塊裡：

```rust
{{#include ../../examples/03_01_async_await/src/lib.rs:static_future_with_borrow}}
```

透過將引數移動到該 `async` 區塊，我們延長了引數的生命週期，使其匹配 `foo` 回傳的 `Future`。

## `async move`

`async` 區塊（block） 與閉包（closure）可以使用 `move` 關鍵字，行為更類似一般的閉包。一個 `async move` block 會取得它引用到的變數之所有權，這些變數就可以活過（outlive）當前的作用範圍（scope），但也就得與放棄其他程式碼共享這些變數的好處。

```rust
{{#include ../../examples/03_01_async_await/src/lib.rs:async_move_examples}}
```

## 在多執行緒的執行器上 `.await`

請注意，當使用多執行緒的 `Future` 執行器時，因為 `.await` 可能導致環境切換至新執行緒，讓 `Future` 可能在不同執行緒間移動，所以任何用在 `async` 裡的變數都必須能在執行緒間傳輸。

這代表使用 `Rc`、`&RefCell` 或其他沒有實作 `Send` trait 的型別，包含沒實作 `Sync` trait 引用型別，都不安全。

（警告：只要不在呼叫 `.await` 的作用域裡，這些型別還是可以使用）

同樣地，在 `.await` 之間持有傳統非 future 的鎖不是個好主意，它可能會造成執行緒池（threadpool）完全鎖上：一個任務拿走了鎖，並且 `.await` 將控制權轉移至執行器，讓其他任務嘗試取得鎖，然後就造成死鎖（deadlock）。我們可以使用 `futures::lock` 而不是 `std::sync` 中的 `Mutex` 從而避免這件事。

[the first chapter]: ../01_getting_started/04_async_await_primer.md
