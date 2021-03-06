# Закрепление (pinning)

Чтобы опросить `futures`, они должны быть закреплены с помощью специального типа под названием
`Pin<T>`. Если Вы прочитаете описание [ типажа `Future` ](../02_execution/02_future.md) в
предыдущем разделе [ "выполнение `Futures` и задач"](../02_execution/01_chapter.md), вы узнаете о
`Pin` из `self: Pin<&mut Self>` в методе`Future:poll`.
Но что это значит, и зачем нам это нужно?

## Для чего перемещение

Закрепление даёт гарантию того, что объект не будет перемещён.
Чтобы понять почему это важно, нам надо помнить как работает `async`/`.await`. 
Рассмотрим следующий код:

```rust
let fut_one = ...;
let fut_two = ...;
async move {
    fut_one.await;
    fut_two.await;
}
```

Под капотом, он создаёт два анонимных типа, которые реализуют типаж `Future`,
предоставляющий метод `poll`, выглядящий примерно так:

```rust
// Тип `Future`, созданный нашим `async { ... }` блоком
struct AsyncFuture {
    fut_one: FutOne,
    fut_two: FutTwo,
    state: State,
}

// Список возможных состояний нашего `async` блока
enum State {
    AwaitingFutOne,
    AwaitingFutTwo,
    Done,
}

impl Future for AsyncFuture {
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self.state {
                State::AwaitingFutOne => match self.fut_one.poll(..) {
                    Poll::Ready(()) => self.state = State::AwaitingFutTwo,
                    Poll::Pending => return Poll::Pending,
                }
                State::AwaitingFutTwo => match self.fut_two.poll(..) {
                    Poll::Ready(()) => self.state = State::Done,
                    Poll::Pending => return Poll::Pending,
                }
                State::Done => return Poll::Ready(()),
            }
        }
    }
}
```

Когда `poll` вызывается первый раз, он опрашивает 
`fut_one`. Если `fut_one` не завершена, 
возвращается `AsyncFuture::poll`. Следующие вызовы 
`poll` будут начинаться там, где завершился 
предыдущий вызов. Этот процесс продолжается до тех пор, пока 
`future` не сможет завершиться.

Однако, что будет, если `async` блок использует ссылки?
Например:

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

Во что скомпилируется эта структура?

```rust
struct ReadIntoBuf<'a> {
    buf: &'a mut [u8], // указывает на `x` далее
}

struct AsyncFuture {
    x: [u8; 128],
    read_into_buf_fut: ReadIntoBuf<'?>, // какое тут время жизни?
}
```

Здесь `future` `ReadIntoBuf` содержит ссылку на другое 
поле нашей структуры, `x`. Однако, если 
`AsyncFuture` будет перемещена, положение 
`x` тоже будет изменено, что сделает указатель, 
сохранённый в `read_into_buf_fut.buf`, недействительным.

Закрепление future в определённом месте памяти, предотвращает 
эту проблему, делая безопасным создание ссылок на данные за 
пределами `async` блока.

## Как использовать закрепление

Тип `Pin` оборачивает указатель на другие типы, 
гарантируя, что значение за указателем не будет перемещено. 
Например, `Pin<&mut T>`, `Pin<&T>`,
`Pin<Box<T>>` - все гарантируют, что положение 
`T` останется неизменным.

У большинства типов нет проблем с перемещением. Эти типы 
реализуют типаж `Unpin`. Указатели на 
`Unpin`-типы могут свободно помещаться в 
`Pin` или извлекаться из него. Например, тип 
`u8` реализует `Unpin`, таким образом 
`Pin<&mut T>` ведёт себя также, как и 
`&mut T`.

Некоторые функции требуют, чтобы `future`, с которыми они 
работают, были `Unpin`. Для использования 
`Future` или `Stream`, не реализующего 
`Unpin`, с функцией, требующей 
`Unpin`-типа, вы сначала должны закрепить значение 
при помощи `Box::pin` (для создания 
`Pin<Box<T>>`) или макроса 
`pin_utils::pin_mut!` (для создания 
`Pin<&mut T>`). Оба `Pin<Box<Fut>>` и 
`Pin<&mut Fut>` могут использоваться как` future`, и 
оба реализуют `Unpin`.

Например:

```rust
use pin_utils::pin_mut; // `pin_utils` - удобный пакет, доступный на crates.io

// Функция, принимающая `Future`, которая реализует `Unpin`.
fn execute_unpin_future(x: impl Future<Output = ()> + Unpin) { ... }

let fut = async { ... };
execute_unpin_future(fut); // Ошибка: `fut` не реализует типаж `Unpin`

// Закрепление с помощью `Box`:
let fut = async { ... };
let fut = Box::pin(fut);
execute_unpin_future(fut); // OK

// Закрепление с помощью `pin_mut!`:
let fut = async { ... };
pin_mut!(fut);
execute_unpin_future(fut); // OK
```
