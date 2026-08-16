# 11. Streams и AsyncIterator 🟡

> **Чему вы научитесь:**
> - Трейту `Stream`: асинхронной итерации по множеству значений
> - Способам создания стримов: `stream::iter`, `async_stream`, `unfold`
> - Комбинаторам стримов: `map`, `filter`, `buffer_unordered`, `fold`
> - Асинхронным трейтам ввода-вывода: `AsyncRead`, `AsyncWrite`, `AsyncBufRead`

## Обзор трейта Stream

Инструмент `Stream` (поток данных) относится к `Iterator` точно так же, как `Future` к одиночному значению — он асинхронно возвращает последовательность элементов:

```rust
// std::iter::Iterator (синхронный, много значений)
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// futures::Stream (асинхронный, много значений)
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

```mermaid
graph LR
    subgraph "Синхронный"
        VAL["Значение<br/>(T)"]
        ITER["Итератор<br/>(много T)"]
    end

    subgraph "Асинхронный"
        FUT["Future<br/>(асинхронное T)"]
        STREAM["Stream<br/>(асинхронно много T)"]
    end

    VAL -->|"сделать асинхронным"| FUT
    ITER -->|"сделать асинхронным"| STREAM
    VAL -->|"сделать множественным"| ITER
    FUT -->|"сделать множественным"| STREAM

    style VAL fill:#e3f2fd,color:#000
    style ITER fill:#e3f2fd,color:#000
    style FUT fill:#c8e6c9,color:#000
    style STREAM fill:#c8e6c9,color:#000
```

### Создание стримов

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{interval, Duration};
use tokio_stream::wrappers::IntervalStream;

// 1. Из итератора
let s = stream::iter(vec![1, 2, 3]);

// 2. Из асинхронного генератора (используя библиотеку async-stream)
// Cargo.toml: async-stream = "0.3"
use async_stream::stream;

fn countdown(from: u32) -> impl futures::Stream<Item = u32> {
    stream! {
        for i in (0..=from).rev() {
            tokio::time::sleep(Duration::from_millis(500)).await;
            yield i;
        }
    }
}

// 3. Из интервала tokio
let tick_stream = IntervalStream::new(interval(Duration::from_secs(1)));

// 4. Из получателя канала (tokio_stream::wrappers)
let (tx, rx) = tokio::sync::mpsc::channel::<String>(100);
let rx_stream = tokio_stream::wrappers::ReceiverStream::new(rx);

// 5. С помощью функции unfold (генерация из асинхронного состояния)
let s = stream::unfold(0u32, |state| async move {
    if state >= 5 {
        None // Стрим завершается
    } else {
        let next = state + 1;
        Some((state, next)) // возвращаем state через yield, новым состоянием становится next
    }
});
```

### Потребление стримов

```rust
use futures::stream::{self, StreamExt};

async fn stream_examples() {
    let s = stream::iter(vec![1, 2, 3, 4, 5]);

    // for_each — обработка каждого элемента
    s.for_each(|x| async move {
        println!("{x}");
    }).await;

    // map + collect
    let doubled: Vec<i32> = stream::iter(vec![1, 2, 3])
        .map(|x| x * 2)
        .collect()
        .await;

    // filter
    let evens: Vec<i32> = stream::iter(1..=10)
        .filter(|x| futures::future::ready(x % 2 == 0))
        .collect()
        .await;

    // buffer_unordered — конкурентная обработка N элементов
    let results: Vec<_> = stream::iter(vec!["url1", "url2", "url3"])
        .map(|url| async move {
            // Имитация HTTP-запроса
            tokio::time::sleep(Duration::from_millis(100)).await;
            format!("ответ от {url}")
        })
        .buffer_unordered(10) // До 10 параллельных запросов
        .collect()
        .await;

    // take, skip, zip, chain — аналогично обычному Iterator
    let first_three: Vec<i32> = stream::iter(1..=100)
        .take(3)
        .collect()
        .await;
}
```

### Сравнение с IAsyncEnumerable из C#

| Возможность | Stream в Rust | IAsyncEnumerable<T> в C# |
|---------|--------------|--------------------------|
| **Синтаксис** | `stream! { yield x; }` | `await foreach` / `yield return` |
| **Отмена** | Сброс (drop) стрима | `CancellationToken` |
| **Обратное давление** | Потребитель контролирует частоту poll | Потребитель контролирует `MoveNextAsync` |
| **Встроенный** | Нет (требуется библиотека `futures`) | Да (начиная с C# 8.0) |
| **Комбинаторы** | `.map()`, `.filter()`, `.buffer_unordered()` | LINQ + `System.Linq.Async` |
| **Обработка ошибок** | `Stream<Item = Result<T, E>>` | Исключение (throw) в асинхронном итераторе |

```rust
// Пример на Rust: стрим строк базы данных
// ПРИМЕЧАНИЕ: try_stream! (а не stream!) необходим при использовании оператора ? внутри блока.
// stream! не пробрасывает ошибки наружу — try_stream! возвращает Err(e) и завершает работу.
fn get_users(db: &Database) -> impl Stream<Item = Result<User, DbError>> + '_ {
    try_stream! {
        let mut cursor = db.query("SELECT * FROM users").await?;
        while let Some(row) = cursor.next().await {
            yield User::from_row(row?);
        }
    }
}

// Потребление:
let mut users = pin!(get_users(&db));
while let Some(result) = users.next().await {
    match result {
        Ok(user) => println!("{}", user.name),
        Err(e) => eprintln!("Ошибка: {e}"),
    }
}
```

```csharp
// Эквивалент на C#:
async IAsyncEnumerable<User> GetUsers() {
    await using var reader = await db.QueryAsync("SELECT * FROM users");
    while (await reader.ReadAsync()) {
        yield return User.FromRow(reader);
    }
}

// Потребление:
await foreach (var user in GetUsers()) {
    Console.WriteLine(user.Name);
}
```

<details>
<summary><strong>🏋️ Упражнение: Создание асинхронного агрегатора статистики</strong> (нажмите для раскрытия)</summary>

**Задача**: Получив стрим показаний датчиков `Stream<Item = f64>`, напишите асинхронную функцию, которая считывает его элементы и возвращает `(count, min, max, average)`. Используйте комбинаторы `StreamExt` вместо банального вызова `.collect()` в `Vec`.

*Hint*: Используйте метод `.fold()`, чтобы накапливать состояние по ходу чтения стрима.

<details>
<summary>🔑 Решение</summary>

```rust
use futures::stream::{self, StreamExt};

#[derive(Debug)]
struct Stats {
    count: usize,
    min: f64,
    max: f64,
    sum: f64,
}

impl Stats {
    fn average(&self) -> f64 {
        if self.count == 0 { 0.0 } else { self.sum / self.count as f64 }
    }
}

async fn compute_stats<S: futures::Stream<Item = f64>>(stream: S) -> Stats {
    stream
        .fold(
            Stats { count: 0, min: f64::INFINITY, max: f64::NEG_INFINITY, sum: 0.0 },
            |mut acc, value| async move {
                acc.count += 1;
                acc.min = acc.min.min(value);
                acc.max = acc.max.max(value);
                acc.sum += value;
                acc
            },
        )
        .await
}

#[tokio::test]
async fn test_stats() {
    let readings = stream::iter(vec![23.5, 24.1, 22.8, 25.0, 23.9]);
    let stats = compute_stats(readings).await;

    assert_eq!(stats.count, 5);
    assert!((stats.min - 22.8).abs() < f64::EPSILON);
    assert!((stats.max - 25.0).abs() < f64::EPSILON);
    assert!((stats.average() - 23.86).abs() < 0.01);
}
```

**Ключевой вывод**: Комбинаторы стримов вроде `.fold()` обрабатывают элементы поочередно, не загружая их все одновременно в память. Это критически важно для работы с большими или бесконечными потоками данных.

</details>
</details>

### Асинхронные трейты ввода-вывода: AsyncRead, AsyncWrite, AsyncBufRead

Точно так же, как `std::io::Read`/`Write` служат основой синхронного ввода-вывода, их асинхронные аналоги формируют базис для асинхронного ввода-вывода. Эти трейты предоставляются модулем `tokio::io` (или `futures::io` для независимого от рантайма кода):

```rust
// tokio::io — асинхронные версии трейтов std::io

/// Асинхронное чтение байт из источника
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,  // Безопасная обертка Tokio вокруг неинициализированной памяти
    ) -> Poll<io::Result<()>>;
}

/// Асинхронная запись байт в приемник
pub trait AsyncWrite {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<io::Result<usize>>;

    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
    fn poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
}

/// Буферизованное чтение с поддержкой строк
pub trait AsyncBufRead: AsyncRead {
    fn poll_fill_buf(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<&[u8]>>;
    fn consume(self: Pin<&mut Self>, amt: usize);
}
```

**На практике** вы редко вызываете эти методы `poll_*` напрямую. Вместо этого вы используете extension-трейты `AsyncReadExt` и `AsyncWriteExt`, предоставляющие удобные вспомогательные методы, поддерживающие `.await`:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt};
use tokio::net::TcpStream;
use tokio::io::BufReader;

async fn io_examples() -> tokio::io::Result<()> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    // AsyncWriteExt: write_all, write_u32, write_buf и т.д.
    stream.write_all(b"GET / HTTP/1.0\r\n\r\n").await?;

    // AsyncReadExt: read, read_exact, read_to_end, read_to_string
    let mut response = Vec::new();
    stream.read_to_end(&mut response).await?;

    // AsyncBufReadExt: read_line, lines(), split()
    let file = tokio::fs::File::open("config.txt").await?;
    let reader = BufReader::new(file);
    let mut lines = reader.lines();
    while let Some(line) = lines.next_line().await? {
        println!("{line}");
    }

    Ok(())
}
```

**Реализация кастомного асинхронного ввода-вывода** — оборачивание протокола поверх сырого TCP-соединения:

```rust
use tokio::io::{AsyncRead, AsyncWrite, ReadBuf};
use std::pin::Pin;
use std::task::{Context, Poll};

/// Протокол с префиксом длины: [u32 длина][байт полезной нагрузки]
struct FramedStream<T> {
    inner: T,
}

impl<T: AsyncRead + AsyncReadExt + Unpin> FramedStream<T> {
    /// Чтение одного целого фрейма
    async fn read_frame(&mut self) -> tokio::io::Result<Vec<u8>> {
        // Считываем 4-байтовый префикс длины
        let len = self.inner.read_u32().await? as usize;

        // Считываем ровно указанное количество байт
        let mut payload = vec![0u8; len];
        self.inner.read_exact(&mut payload).await?;
        Ok(payload)
    }
}

impl<T: AsyncWrite + AsyncWriteExt + Unpin> FramedStream<T> {
    /// Запись одного целого фрейма
    async fn write_frame(&mut self, data: &[u8]) -> tokio::io::Result<()> {
        self.inner.write_u32(data.len() as u32).await?;
        self.inner.write_all(data).await?;
        self.inner.flush().await?;
        Ok(())
    }
}
```

| Синхронный трейт | Асинхронный трейт (tokio) | Асинхронный трейт (futures) | Extension-трейт |
|-----------|--------------------|-----------------------|----------------|
| `std::io::Read` | `tokio::io::AsyncRead` | `futures::io::AsyncRead` | `AsyncReadExt` |
| `std::io::Write` | `tokio::io::AsyncWrite` | `futures::io::AsyncWrite` | `AsyncWriteExt` |
| `std::io::BufRead` | `tokio::io::AsyncBufRead` | `futures::io::AsyncBufRead` | `AsyncBufReadExt` |
| `std::io::Seek` | `tokio::io::AsyncSeek` | `futures::io::AsyncSeek` | `AsyncSeekExt` |

> **Трейты ввода-вывода tokio по сравнению с futures**: Они похожи, но не идентичны — `AsyncRead` в Tokio использует тип `ReadBuf` (который безопасно обрабатывает неинициализированную память), в то время как `futures::AsyncRead` работает с `&mut [u8]`. Для восстановления совместимости используйте `tokio_util::compat`.

> **Утилиты копирования**: Функция `tokio::io::copy(&mut reader, &mut writer)` — это асинхронный эквивалент `std::io::copy`. Она крайне полезна для прокси-серверов или передачи файлов. Функция `tokio::io::copy_bidirectional` выполняет копирование в обоих направлениях одновременно.

<details>
<summary><strong>🏋️ Упражнение: Создание асинхронного счетчика строк</strong> (нажмите для раскрытия)</summary>

**Задача**: Напишите асинхронную функцию, которая принимает любой источник `AsyncBufRead` и возвращает количество непустых строк. Она должна работать с файлами, TCP-соединениями или любыми буферизованными читателями.

*Hint*: Используйте метод `AsyncBufReadExt::lines()` и считайте строки, для которых `!line.is_empty()`.

<details>
<summary>🔑 Решение</summary>

```rust
use tokio::io::AsyncBufReadExt;

async fn count_non_empty_lines<R: tokio::io::AsyncBufRead + Unpin>(
    reader: R,
) -> tokio::io::Result<usize> {
    let mut lines = reader.lines();
    let mut count = 0;
    while let Some(line) = lines.next_line().await? {
        if !line.is_empty() {
            count += 1;
        }
    }
    Ok(count)
}

// Работает с любым типом, реализующим AsyncBufRead:
// let file = tokio::io::BufReader::new(tokio::fs::File::open("data.txt").await?);
// let count = count_non_empty_lines(file).await?;
//
// let tcp = tokio::io::BufReader::new(TcpStream::connect("...").await?);
// let count = count_non_empty_lines(tcp).await?;
```

**Ключевой вывод**: Программируя под абстрактный трейт `AsyncBufRead` вместо конкретного типа, вы делаете свой код ввода-вывода переиспользуемым для файлов, сокетов, пайпов и даже буферов в оперативной памяти (`tokio::io::BufReader::new(std::io::Cursor::new(data))`).

</details>
</details>

> **Основные выводы — Стримы и AsyncIterator**
> - `Stream` — это асинхронный эквивалент обычного `Iterator`. Он возвращает `Poll::Ready(Some(item))` или `Poll::Ready(None)`
> - Метод `.buffer_unordered(N)` обрабатывает N элементов стрима конкурентно — это главный инструмент обеспечения конкурентности в стримах
> - Макрос `async_stream::stream!` — самый простой способ написания кастомных стримов (использует оператор `yield`)
> - Трейты `AsyncRead` / `AsyncBufRead` позволяют писать обобщенный и переиспользуемый код ввода-вывода для файлов, сокетов и пайпов

> **См. также:** [Глава 9 — Когда Tokio не подходит](ch09-when-tokio-isnt-the-right-fit.md) для изучения `FuturesUnordered` (близкий по духу паттерн), [Глава 13 — Паттерны для продакшена](ch13-production-patterns.md) для настройки обратного давления с помощью каналов ограниченного размера

***
