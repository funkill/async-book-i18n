# Типаж `Stream`

Типаж `Stream` похож на `Future`, но может давать несколько значений до завершения, а также похож на типаж `Iterator` из стандартной библиотеки:

```rust
{{#include ../../examples/05_01_streams/src/lib.rs:stream_trait}}
```

Одним из распространённых примеров `Stream` является `Receiver` для типа канала из
пакета `futures`. Это даёт `Some(val)` каждый раз, когда значение отправляется
от `Sender`, и даст `None` после того, как `Sender` был удалён из памяти и все ожидающие сообщения были получены:

```rust
{{#include ../../examples/05_01_streams/src/lib.rs:channels}}
```
