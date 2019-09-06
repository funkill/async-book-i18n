# 迭代與並行

和同步的 `Iterator` 一樣，迭代並處理在 `Steam` 裡的值有許多作法。有許多組合子（combinator）風格的方法，例如 `map`、`filter` 和 `fold`，和它們*遇錯就提前退出*的表親 `try_map`、`try_filter` 以及 `try_fold`。

不幸的是 `for` 迴圈無法配合 `Stream` 使用，但命令式風格程式碼 `while let` 與 `next`/`try_next` 可以使用：

```rust
{{#include ../../examples/05_02_iteration_and_concurrency/src/lib.rs:nexts}}
```

不過，如果一次只處理一個元素，那就錯失善用並行的良機，畢竟並行是寫非同步程式碼的初衷嘛。想要並行處理 steam 上多個元素，請用 `for_each_concurrent` 與 `try_for_each_concurrent` 方法：

```rust
{{#include ../../examples/05_02_iteration_and_concurrency/src/lib.rs:try_for_each_concurrent}}
```
