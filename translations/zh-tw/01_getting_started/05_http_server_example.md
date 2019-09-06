# 案例：HTTP 伺服器

我們用 `async/.await` 打造一個 echo 伺服器吧！

首先，執行 `rustup update nightly` 讓你去的最新最棒的 Rust，我們正使用最前沿的功能，確保最新版本非常重要。當你完成上述指令，執行 `cargo +nightly new async-await-echo` 來建立新專案，並開啟 `async-await-echo` 資料夾。

讓我們在 `Cargo.toml` 中加上一些相依函式庫：

```toml
{{#include ../../examples/01_05_http_server/Cargo.toml:9:18}}
```

現在已經取得我們的相依函式庫，可以開始寫程式了。開啟 `src/main.rs` 並在檔案最上面開啟 `async_await` 功能：

```rust
#![feature(async_await)]
```

這將會加入 nightly 版專屬但很快就會穩定的 `async/await` 語法支援。

此外，有些 import 需要新增：

```rust
{{#include ../../examples/01_05_http_server/src/lib.rs:imports}}
```

當這些 imports 加入後，將這些鍋碗瓢盆放在一起，就能開始接收請求：

```rust
{{#include ../../examples/01_05_http_server/src/lib.rs:boilerplate}}
```

如果現在跑 `cargo run`，應該會看到「Listening on http://127.0.0.1:3000」輸出在終端機上。若在你的瀏覽器開啟這個鏈結，會看到「hello, world」出現在你的瀏覽器。恭喜！你剛寫下第一個 Rust 非同步網頁伺服器。

你也可以檢視這個請求本身，會包含譬如請求的 URI、HTTP 版本、headers、以及其他詮釋資料。舉例來說，你可以打印出請求的 URI：

```rust
println!("Got request at {:?}", req.uri());
```

你可能注意到我們尚未做任何非同步的事情來處理這個請求，我們就只是即刻回應罷了，所以我們並無善用 `async fn` 給予我們的彈性。與其只返回一個靜態訊息，讓我們使用 Hyper 的 HTTP 客戶端來代理使用者的請求到其他網站。

我們從解析想要請求的 URL 開始：

```rust
{{#include ../../examples/01_05_http_server/src/lib.rs:parse_url}}
```

接下來我們建立一個新的 `hyper::Client` 並用之生成一個 `GET` 請求，這個請求會回傳一個回應給使用者。

```rust
{{#include ../../examples/01_05_http_server/src/lib.rs:get_request}}
```

`Client::get` 返回一個 `hyper::client::FutureResponse`，這個 future 實作了 `Future<Output = Result<Response, Error>>`（或 futures 0.1 的 `Future<Item = Response, Error = Error>`）。當我們 `.await` 這個 future，將會發送一個 HTTP 請求，當前的任務會暫時停止（suspend），而這個任務會進入佇列中，在收到回應後繼續執行。

現在，執行 `cargo run` 並在瀏覽器中打開 `http://127.0.0.1:3000/foo`，會看到 Rust 首頁，以及以下的終端機輸出：

```
Listening on http://127.0.0.1:3000
Got request at /foo
making request to http://www.rust-lang.org/en-US/
request finished-- returning response
```

恭喜呀！你剛剛代理了一個 HTTP 請求。
