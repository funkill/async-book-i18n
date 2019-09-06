# 同時執行多個 Future

至此，我們大部分都是使用 `.await` 來執行 future，並在完成特定的 `Future` 前，阻塞當前的任務。不過，真實的非同步應用程式通常需要並行處理多個不同的操作。

本章涵蓋一些同時執行多個非同步操作的方法：

- `join!`：等待所有 future 完成
- `select!`：等待多個 future 中的其中一個完成
- Spawning：建立一個最上層的任務，並在當前環境中執行該 future 到完成
- `FuturesUnordered`：一群會產出每個 subfuture 結果的 future
