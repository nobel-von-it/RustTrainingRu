# 15. Упражнения

### Упражнение 1: Асинхронный эхо-сервер

Напишите TCP эхо-сервер, который обрабатывает подключения множества клиентов конкурентно.

**Требования**:
- Слушать порт `127.0.0.1:8080`
- Принимать входящие соединения и отправлять обратно каждую полученную строку
- Корректно обрабатывать отключения клиентов
- Логировать моменты подключения и отключения клиентов

<details>
<summary>🔑 Решение</summary>

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Эхо-сервер запущен на :8080");

    loop {
        let (socket, addr) = listener.accept().await?;
        println!("[{addr}] Подключен");

        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut reader = BufReader::new(reader);
            let mut line = String::new();

            loop {
                line.clear();
                match reader.read_line(&mut line).await {
                    Ok(0) => {
                        println!("[{addr}] Отключен");
                        break;
                    }
                    Ok(_) => {
                        print!("[{addr}] Эхо: {line}");
                        if writer.write_all(line.as_bytes()).await.is_err() {
                            println!("[{addr}] Ошибка записи, отключение");
                            break;
                        }
                    }
                    Err(e) => {
                        eprintln!("[{addr}] Ошибка чтения: {e}");
                        break;
                    }
                }
            }
        });
    }
}
```

</details>

---

### Упражнение 2: Конкурентный загрузчик URL-адресов с ограничением частоты

Скачайте список URL-адресов конкурентно, при этом одновременно должно выполняться не более 5 запросов.

<details>
<summary>🔑 Решение</summary>

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

async fn fetch_urls(urls: Vec<String>) -> Vec<Result<String, String>> {
    // buffer_unordered(5) гарантирует, что одновременно будут опрашиваться не более 5 фьючеров
    // дополнительный семафор здесь не требуется.
    let results: Vec<_> = stream::iter(urls)
        .map(|url| {
            async move {
                println!("Загрузка: {url}");

                match reqwest::get(&url).await {
                    Ok(resp) => match resp.text().await {
                        Ok(body) => Ok(body),
                        Err(e) => Err(format!("{url}: {e}")),
                    },
                    Err(e) => Err(format!("{url}: {e}")),
                }
            }
        })
        .buffer_unordered(5) // ← Один лишь этот вызов ограничивает конкурентность числом 5
        .collect()
        .await;

    results
}

// ПРИМЕЧАНИЕ: Используйте Semaphore, когда нужно ограничить уровень конкурентности между
// независимо запущенными задачами (tokio::spawn). Используйте buffer_unordered
// при обработке стрима. Не комбинируйте оба подхода для одного и того же лимита.
```

</details>

---

### Упражнение 3: Плавное завершение работы пула рабочих задач

Постройте обработчик задач, содержащий:
- Очередь задач на основе каналов
- N рабочих задач (workers), вычитывающих элементы из очереди
- Плавное завершение работы при нажатии Ctrl+C: прекратить прием новых задач, завершить уже взятые в обработку

<details>
<summary>🔑 Решение</summary>

```rust
use tokio::sync::{mpsc, watch};
use tokio::time::{sleep, Duration};

struct WorkItem {
    id: u64,
    payload: String,
}

#[tokio::main]
async fn main() {
    let (work_tx, work_rx) = mpsc::channel::<WorkItem>(100);
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // Запускаем 4 рабочих
    let mut worker_handles = Vec::new();
    let work_rx = std::sync::Arc::new(tokio::sync::Mutex::new(work_rx));

    for id in 0..4 {
        let rx = work_rx.clone();
        let mut shutdown = shutdown_rx.clone();
        let handle = tokio::spawn(async move {
            loop {
                let item = {
                    let mut rx = rx.lock().await;
                    tokio::select! {
                        item = rx.recv() => item,
                        _ = shutdown.changed() => {
                            if *shutdown.borrow() { None } else { continue }
                        }
                    }
                };

                match item {
                    Some(work) => {
                        println!("Worker {id}: processing item {}", work.id);
                        sleep(Duration::from_millis(200)).await; // Имитируем работу
                        println!("Worker {id}: done with item {}", work.id);
                    }
                    None => {
                        println!("Worker {id}: channel closed, exiting");
                        break;
                    }
                }
            }
        });
        worker_handles.push(handle);
    }

    // Производитель: отправляем несколько задач
    let producer = tokio::spawn(async move {
        for i in 0..20 {
            let _ = work_tx.send(WorkItem {
                id: i,
                payload: format!("task-{i}"),
            }).await;
            sleep(Duration::from_millis(50)).await;
        }
    });

    // Ждем нажатия Ctrl+C
    tokio::signal::ctrl_c().await.unwrap();
    println!("\nПолучен сигнал завершения работы!");
    shutdown_tx.send(true).unwrap();
    producer.abort(); // Отменяем задачу-производитель

    // Ждем завершения рабочих задач
    for handle in worker_handles {
        let _ = handle.await;
    }
    println!("Все рабочие завершили работу. До свидания!");
}
```

</details>

---

### Упражнение 4: Создание простого асинхронного мьютекса с нуля

Реализуйте асинхронный мьютекс, используя каналы/семафоры (без использования готового `tokio::sync::Mutex`).

*Hint*: Используйте `tokio::sync::Semaphore` с 1 разрешением (permit) для сериализации доступа.

<details>
<summary>🔑 Решение</summary>

```rust
use std::cell::UnsafeCell;
use std::sync::Arc;
use tokio::sync::{OwnedSemaphorePermit, Semaphore};

pub struct SimpleAsyncMutex<T> {
    data: Arc<UnsafeCell<T>>,
    semaphore: Arc<Semaphore>,
}

// SAFETY: Доступ к T сериализован семафором (максимум 1 разрешение).
unsafe impl<T: Send> Send for SimpleAsyncMutex<T> {}
unsafe impl<T: Send> Sync for SimpleAsyncMutex<T> {}

pub struct SimpleGuard<T> {
    data: Arc<UnsafeCell<T>>,
    _permit: OwnedSemaphorePermit, // Удаляется при сбросе гарда → освобождает блокировку
}

impl<T> SimpleAsyncMutex<T> {
    pub fn new(value: T) -> Self {
        SimpleAsyncMutex {
            data: Arc::new(UnsafeCell::new(value)),
            semaphore: Arc::new(Semaphore::new(1)),
        }
    }

    pub async fn lock(&self) -> SimpleGuard<T> {
        let permit = self.semaphore.clone().acquire_owned().await.unwrap();
        SimpleGuard {
            data: self.data.clone(),
            _permit: permit,
        }
    }
}

impl<T> std::ops::Deref for SimpleGuard<T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: Мы удерживаем единственное разрешение семафора, поэтому другие
        // SimpleGuard не существуют → гарантирован эксклюзивный доступ.
        unsafe { &*self.data.get() }
    }
}

impl<T> std::ops::DerefMut for SimpleGuard<T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: Та же логика — одно разрешение гарантирует эксклюзивность.
        unsafe { &mut *self.data.get() }
    }
}

// При сбросе SimpleGuard гард _permit уничтожается, освобождая разрешение семафора —
// другой вызов lock() может быть выполнен.

// Использование:
// let mutex = SimpleAsyncMutex::new(vec![1, 2, 3]);
// {
//     let mut guard = mutex.lock().await;
//     guard.push(4);
// } // разрешение освобождается здесь
```

**Ключевой вывод**: Асинхронные мьютексы обычно строятся поверх семафоров. Семафор предоставляет асинхронный механизм ожидания: если блокировка удерживается, вызов `acquire()` приостанавливает задачу до тех пор, пока разрешение не будет освобождено. Именно так устроен `tokio::sync::Mutex` внутри.

> **Почему `UnsafeCell`, а не `std::sync::Mutex`?** Предыдущая версия этого упражнения использовала `Arc<Mutex<T>>`, где `Deref`/`DerefMut` вызывали `.lock().unwrap()`. Это не скомпилируется: возвращаемая ссылка `&T` заимствуется из временного `MutexGuard`, который уничтожается сразу же. `UnsafeCell` позволяет избежать промежуточного гарда, а сериализация на базе семафора делает такой `unsafe` корректным.

</details>

---

### Упражнение 5: Конвейер стрима

Постройте конвейер обработки данных с помощью стримов:
1. Генерировать числа от 1 до 100
2. Фильтровать только четные числа
3. Возводить каждое число в квадрат
4. Обрабатывать по 10 элементов одновременно (симулировать работу через сон)
5. Собрать результаты

<details>
<summary>🔑 Решение</summary>

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let results: Vec<u64> = stream::iter(1u64..=100)
        // Шаг 2: Фильтруем четные
        .filter(|x| futures::future::ready(x % 2 == 0))
        // Шаг 3: Возводим в квадрат
        .map(|x| x * x)
        // Шаг 4: Обрабатываем конкурентно (имитируем асинхронную работу)
        .map(|x| async move {
            sleep(Duration::from_millis(50)).await;
            println!("Обработано: {x}");
            x
        })
        .buffer_unordered(10) // по 10 одновременно
        // Шаг 5: Собираем результаты
        .collect()
        .await;

    println!("Получено {} результатов", results.len());
    println!("Сумма: {}", results.iter().sum::<u64>());
}
```

</details>

---

### Упражнение 6: Реализация Select с таймаутом

Без использования макросов `tokio::select!` или `tokio::time::timeout` реализуйте структуру/функцию, которая устраивает гонку фьючера с дедлайном и возвращает `Either::Left(result)` при успехе или `Either::Right(())` при таймауте.

*Hint*: Опирайтесь на структуру `Select` из Главы 6 и `TimerFuture` из той же главы.

<details>
<summary>🔑 Решение</summary>

```rust,ignore
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

pub enum Either<A, B> {
    Left(A),
    Right(B),
}

pub struct Timeout<F> {
    future: F,
    timer: TimerFuture, // Из Главы 6
}

impl<F: Future + Unpin> Timeout<F> {
    pub fn new(future: F, duration: Duration) -> Self {
        Timeout {
            future,
            timer: TimerFuture::new(duration),
        }
    }
}

impl<F: Future + Unpin> Future for Timeout<F> {
    type Output = Either<F::Output, ()>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // Проверяем, завершен ли основной фьючер
        if let Poll::Ready(val) = Pin::new(&mut self.future).poll(cx) {
            return Poll::Ready(Either::Left(val));
        }

        // Проверяем, истек ли таймер
        if let Poll::Ready(()) = Pin::new(&mut self.timer).poll(cx) {
            return Poll::Ready(Either::Right(()));
        }

        Poll::Pending
    }
}

// Использование:
// match Timeout::new(fetch_data(), Duration::from_secs(5)).await {
//     Either::Left(data) => println!("Получены данные: {data}"),
//     Either::Right(()) => println!("Таймаут!"),
// }
```

**Ключевой вывод**: механизмы `select`/`timeout` представляют собой простой опрос двух фьючеров с отслеживанием того, какой из них завершится первым. Вся асинхронная экосистема построена вокруг этих элементарных примитивов: poll, `Pending`/`Ready` и `Waker`.

</details>
```
