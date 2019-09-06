# `Stream` Trait

`Stream` trait 類似於 `Future`，不同的是在完成前可以產生多個值，就如同標準函式庫的 `Iterator` trait：

```rust
{{#include ../../examples/05_01_streams/src/lib.rs:stream_trait}}
```

常見的 `Stream` 範例是 `futures` 模組裡的通道（channel）型別的 `Receiver`。每次有值從 `Sender` 發送端發送，它就會產生 `Some(val)`；或是當整個 `Sender` 被捨棄（dropped）且所有等待中的訊息都被接收到，`Stream` 就會產生 `None`：

```rust
{{#include ../../examples/05_01_streams/src/lib.rs:channels}}
```
