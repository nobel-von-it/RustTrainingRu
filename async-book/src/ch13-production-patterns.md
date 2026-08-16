# 13. Паттерны для продакшена 🔴

> **Чему вы научитесь:**
> - Плавному завершению работы при помощи каналов `watch` и макроса `select!`
> - Обратному давлению (backpressure): каналы ограниченного размера предотвращают OOM (выход за лимиты памяти)
> - Структурированной конкурентности: `JoinSet` и `TaskTracker`
> - Таймаутам, повторным попыткам и экспоненциальной задержке
> - Обработке ошибок: `thiserror` против `anyhow`, паттерну с двойным `?`
> - Tower: паттерну middleware, используемому в axum, tonic и hyper

## Плавное завершение работы (Graceful Shutdown)

Серверы в продакшене должны завершать работу корректно: дорабатывать текущие запросы, сбрасывать буферы на диск и закрывать активные соединения:

```rust
use tokio::signal;
use tokio::sync::watch;

async fn main_server() {
    // Создаем канал для сигнала завершения работы
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // Запускаем сервер в фоновой задаче
    let server_handle = tokio::spawn(run_server(shutdown_rx.clone()));

    // Ждем нажатия Ctrl+C
    signal::ctrl_c().await.expect("Не удалось настроить обработчик Ctrl+C");
    println!("Получен сигнал завершения работы, завершаем обработку активных запросов...");

    // Уведомляем все задачи о завершении
    // ПРИМЕЧАНИЕ: Вызов .unwrap() используется для краткости. Код для продакшена должен
    // обрабатывать случай, когда все получатели были удалены.
    shutdown_tx.send(true).unwrap();

    // Ждем завершения сервера (с таймаутом)
    match tokio::time::timeout(
        std::time::Duration::from_secs(30),
        server_handle,
    ).await {
        Ok(Ok(())) => println!("Сервер успешно завершил работу"),
        Ok(Err(e)) => eprintln!("Ошибка сервера: {e}"),
        Err(_) => eprintln!("Таймаут завершения работы сервера — принудительный выход"),
    }
}

async fn run_server(mut shutdown: watch::Receiver<bool>) {
    loop {
        tokio::select! {
            // Принимаем новые соединения
            conn = accept_connection() => {
                let shutdown = shutdown.clone();
                tokio::spawn(handle_connection(conn, shutdown));
            }
            // Сигнал завершения
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    println!("Прекращаем принимать новые соединения");
                    break;
                }
            }
        }
    }
    // Активные соединения завершатся самостоятельно,
    // так как у них есть собственные клоны shutdown_rx
}

async fn handle_connection(conn: Connection, mut shutdown: watch::Receiver<bool>) {
    loop {
        tokio::select! {
            request = conn.next_request() => {
                // Полностью обрабатываем запрос — не бросаем его на полпути
                process_request(request).await;
            }
            _ = shutdown.changed() => {
                if *shutdown.borrow() {
                    // Завершаем текущий запрос и выходим
                    break;
                }
            }
        }
    }
}
```

```mermaid
sequenceDiagram
    participant OS as Сигнал ОС
    participant Main as Главная задача
    participant WCH as Канал watch
    participant W1 as Рабочий поток 1
    participant W2 as Рабочий поток 2

    OS->>Main: SIGINT (Ctrl+C)
    Main->>WCH: send(true)
    WCH-->>W1: changed()
    WCH-->>W2: changed()

    Note over W1: Завершение текущего запроса
    Note over W2: Завершение текущего запроса

    W1-->>Main: Задача завершена
    W2-->>Main: Задача завершена
    Main->>Main: Все рабочие завершили работу → выход
```

### Обратное давление (Backpressure) с помощью ограниченных каналов

Использование неограниченных (unbounded) каналов может привести к нехватке памяти (OOM), если производитель отправляет сообщения быстрее, чем потребитель успевает их обрабатывать. Всегда используйте в продакшене ограниченные (bounded) каналы:

```rust
use tokio::sync::mpsc;

async fn backpressure_example() {
    // Ограниченный канал: максимум 100 элементов в буфере
    let (tx, mut rx) = mpsc::channel::<WorkItem>(100);

    // Производитель: замедляется естественным образом, когда буфер заполнен
    let producer = tokio::spawn(async move {
        for i in 0..1_000_000 {
            // send() асинхронен — ждет свободного места в буфере
            // Это создает естественное обратное давление (backpressure)!
            tx.send(WorkItem { id: i }).await.unwrap();
        }
    });

    // Потребитель: обрабатывает элементы в своем темпе
    let consumer = tokio::spawn(async move {
        while let Some(item) = rx.recv().await {
            process(item).await; // Медленная обработка — это нормально, производитель подождет
        }
    });

    let _ = tokio::join!(producer, consumer);
}

// Сравните с неограниченным каналом — ОПАСНО:
// let (tx, rx) = mpsc::unbounded_channel(); // Нет обратного давления!
// Производитель может заполнять память бесконечно
```

### Структурированная конкурентность: JoinSet и TaskTracker

Тип `JoinSet` группирует связанные задачи и гарантирует, что они все завершатся:

```rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration};

async fn structured_concurrency() {
    let mut set = JoinSet::new();

    // Запускаем пачку задач
    for url in get_urls() {
        set.spawn(async move {
            fetch_and_process(url).await
        });
    }

    // Собираем все результаты (порядок не гарантируется)
    let mut results = Vec::new();
    while let Some(result) = set.join_next().await {
        match result {
            Ok(Ok(data)) => results.push(data),
            Ok(Err(e)) => eprintln!("Ошибка задачи: {e}"),
            Err(e) => eprintln!("Задача запаниковала: {e}"),
        }
    }

    // Здесь ВСЕ задачи завершены — нет зависшей фоновой работы
    println!("Обработано {} элементов", results.len());
}

// TaskTracker (tokio-util 0.7.9+) — ожидание всех запущенных задач
use tokio_util::task::TaskTracker;

async fn with_tracker() {
    let tracker = TaskTracker::new();

    for i in 0..10 {
        tracker.spawn(async move {
            sleep(Duration::from_millis(10 * i)).await;
            println!("Задача {i} завершена");
        });
    }

    tracker.close(); // Больше задачи добавляться не будут
    tracker.wait().await; // Ждем ВСЕ отслеживаемые задачи
    println!("Все задачи завершены");
}
```

### Таймауты и повторные попытки

```rust
use tokio::time::{timeout, sleep, Duration};

// Простой таймаут
async fn with_timeout() -> Result<Response, Error> {
    match timeout(Duration::from_secs(5), fetch_data()).await {
        Ok(Ok(response)) => Ok(response),
        Ok(Err(e)) => Err(Error::Fetch(e)),
        Err(_) => Err(Error::Timeout),
    }
}

// Повторные попытки с экспоненциальной задержкой
async fn retry_with_backoff<F, Fut, T, E>(
    max_attempts: u32,
    base_delay_ms: u64,
    operation: F,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut delay = Duration::from_millis(base_delay_ms);

    for attempt in 1..=max_attempts {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) => {
                if attempt == max_attempts {
                    eprintln!("Последняя попытка {attempt} провалена: {e}");
                    return Err(e);
                }
                eprintln!("Попытка {attempt} провалена: {e}, повтор через {delay:?}");
                sleep(delay).await;
                delay *= 2; // Экспоненциальная задержка
            }
        }
    }
    unreachable!()
}

// Использование:
// let result = retry_with_backoff(3, 100, || async {
//     reqwest::get("https://api.example.com/data").await
// }).await?;
```

> **Совет для продакшена — добавьте джиттер (jitter)**: Функция выше использует чистую экспоненциальную задержку. Но в реальных условиях при массовом сбое множество клиентов начнут делать повторные попытки в одно и то же время, создавая лавинообразную нагрузку (эффект thundering herd). Добавьте случайное дрожание (jitter) — например, `sleep(delay + rand_jitter)`, где `rand_jitter` выбирается в диапазоне от `0` до `delay/4`, чтобы распределить попытки во времени.

## Обработка ошибок в асинхронном коде

Асинхронность привносит уникальные проблемы при передаче ошибок: запущенные задачи создают границы ошибок, ошибки таймаутов оборачивают внутренние ошибки, а оператор `?` ведет себя иначе, когда фьючеры пересекают границы задач.

**thiserror против anyhow** — выбор правильного инструмента:

```rust
// thiserror: Описание структурированных ошибок для библиотек и публичных API
// Каждый вариант описан явно — пользователи могут делать match по конкретным ошибкам
use thiserror::Error;

#[derive(Error, Debug)]
enum DiagError {
    #[error("IPMI command failed: {0}")]
    Ipmi(#[from] IpmiError),

    #[error("Sensor {sensor} out of range: {value}°C (max {max}°C)")]
    OverTemp { sensor: String, value: f64, max: f64 },

    #[error("Operation timed out after {0:?}")]
    Timeout(std::time::Duration),

    #[error("Task panicked: {0}")]
    TaskPanic(#[from] tokio::task::JoinError),
}

// anyhow: Быстрая обработка ошибок для приложений и прототипов
// Оборачивает любую ошибку — нет необходимости описывать типы на каждый случай
use anyhow::{Context, Result};

async fn run_diagnostics() -> Result<()> {
    let config = load_config()
        .await
        .context("Failed to load diagnostic config")?;  // Добавляет контекст

    let result = run_gpu_test(&config)
        .await
        .context("GPU diagnostic failed")?;              // Добавляет контекст по цепочке

    Ok(())
}
// anyhow выведет: "GPU diagnostic failed: IPMI command failed: timeout"
```

| Библиотека | Когда использовать | Тип ошибки | Проверка совпадения (Matching) |
|-------|----------|-----------|----------|
| `thiserror` | Код библиотек, публичные API | `enum MyError { ... }` | `match err { MyError::Timeout => ... }` |
| `anyhow` | Приложения, CLI-утилиты, скрипты | `anyhow::Error` (тип стерт) | `err.downcast_ref::<MyError>()` |
| Обе вместе | Библиотека использует `thiserror`, приложение оборачивает их в `anyhow` | Лучшее от обоих решений | Ошибки библиотеки структурированы, приложению на это всё равно |

**Паттерн с двойным `?`** при работе с `tokio::spawn`:

```rust
use thiserror::Error;
use tokio::task::JoinError;

#[derive(Error, Debug)]
enum AppError {
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),

    #[error("Task panicked: {0}")]
    TaskPanic(#[from] JoinError),
}

async fn spawn_with_errors() -> Result<String, AppError> {
    let handle = tokio::spawn(async {
        let resp = reqwest::get("https://example.com").await?;
        Ok::<_, reqwest::Error>(resp.text().await?)
    });

    // Двойной ?: первый ? распаковывает JoinError (паника задачи), второй ? распаковывает внутренний Result
    let result = handle.await??;
    Ok(result)
}
```

**Проблема границы ошибок** — `tokio::spawn` стирает контекст:

```rust
// ❌ Контекст ошибки теряется при пересечении границ spawn:
async fn bad_error_handling() -> Result<()> {
    let handle = tokio::spawn(async {
        some_fallible_work().await  // Возвращает Result<T, SomeError>
    });

    // handle.await возвращает Result<Result<T, SomeError>, JoinError>
    // Внутренняя ошибка не содержит контекста о том, какая именно задача упала
    let result = handle.await??;
    Ok(())
}

// ✅ Добавление контекста на границе spawn:
async fn good_error_handling() -> Result<()> {
    let handle = tokio::spawn(async {
        some_fallible_work()
            .await
            .context("worker task failed")  // Контекст перед выходом за границу задачи
    });

    let result = handle.await
        .context("worker task panicked")??;  // Контекст также и для ошибки JoinError
    Ok(())
}
```

**Ошибки таймаутов** — оборачивание против замены:

```rust
use tokio::time::{timeout, Duration};

async fn with_timeout_context() -> Result<String, DiagError> {
    let dur = Duration::from_secs(30);
    match timeout(dur, fetch_sensor_data()).await {
        Ok(Ok(data)) => Ok(data),
        Ok(Err(e)) => Err(e),                      // Внутренняя ошибка сохраняется
        Err(_) => Err(DiagError::Timeout(dur)),     // Таймаут → типизированная ошибка
    }
}
```

### Tower: Паттерн Middleware

Библиотека [Tower](https://docs.rs/tower) определяет компонуемый трейт `Service` — основу асинхронного промежуточного ПО (middleware) в Rust (используется в `axum`, `tonic`, `hyper`):

```rust
// Основной трейт библиотеки Tower (упрощенно):
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

Middleware оборачивает `Service` для реализации сквозной функциональности (логирование, таймауты, ограничение частоты запросов) без изменения внутренней логики самого сервиса:

```rust
use tower::{ServiceBuilder, timeout::TimeoutLayer, limit::RateLimitLayer};
use std::time::Duration;

let service = ServiceBuilder::new()
    .layer(TimeoutLayer::new(Duration::from_secs(10)))       // Внешний слой: таймаут
    .layer(RateLimitLayer::new(100, Duration::from_secs(1))) // Затем: ограничение частоты запросов
    .service(my_handler);                                     // Самый внутренний слой: ваш код
```

**Почему это важно**: Если вы использовали middleware в ASP.NET или Express.js, то Tower — это их аналог на Rust. С его помощью в промышленные сервисы добавляется сквозная функциональность без дублирования кода.

### Упражнение: Плавное завершение работы пула рабочих задач

<details>
<summary>🏋️ Упражнение (нажмите для раскрытия)</summary>

**Задача**: Постройте обработчик задач с очередью на основе каналов, N рабочими задачами (workers) и плавным завершением работы при нажатии Ctrl+C. Рабочие задачи должны закончить выполнение взятых в обработку элементов перед выходом.

<details>
<summary>🔑 Решение</summary>

```rust
use tokio::sync::{mpsc, watch};
use tokio::time::{sleep, Duration};

struct WorkItem { id: u64, payload: String }

#[tokio::main]
async fn main() {
    let (work_tx, work_rx) = mpsc::channel::<WorkItem>(100);
    let (shutdown_tx, shutdown_rx) = watch::channel(false);
    let work_rx = std::sync::Arc::new(tokio::sync::Mutex::new(work_rx));

    let mut handles = Vec::new();
    for id in 0..4 {
        let rx = work_rx.clone();
        let mut shutdown = shutdown_rx.clone();
        handles.push(tokio::spawn(async move {
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
                        println!("Worker {id}: processing {}", work.id);
                        sleep(Duration::from_millis(200)).await;
                    }
                    None => break,
                }
            }
        }));
    }

    // Отправляем задачи
    for i in 0..20 {
        let _ = work_tx.send(WorkItem { id: i, payload: format!("task-{i}") }).await;
        sleep(Duration::from_millis(50)).await;
    }

    // При Ctrl+C: сигнализируем о завершении, ждем завершения рабочих
    // ПРИМЕЧАНИЕ: Вызов .unwrap() используется для краткости — обрабатывайте ошибки в продакшене.
    tokio::signal::ctrl_c().await.unwrap();
    shutdown_tx.send(true).unwrap();
    for h in handles { let _ = h.await; }
    println!("Работа успешно завершена.");
}
```

</details>
</details>

> **Основные выводы — Паттерны для продакшена**
> - Используйте канал `watch` и макрос `select!` для скоординированного плавного завершения работы
> - Ограниченные каналы (`mpsc::channel(N)`) обеспечивают **обратное давление (backpressure)** — отправители приостанавливают работу при переполнении буфера
> - `JoinSet` и `TaskTracker` реализуют **структурированную конкурентность**: отслеживание, отмена и ожидание целых групп задач
> - Всегда добавляйте таймауты к сетевым операциям — `tokio::time::timeout(dur, fut)`
> - Трейт `Service` из библиотеки Tower — стандартный паттерн middleware для промышленных сервисов на Rust

> **См. также:** [Глава 8 — Глубокое погружение в Tokio](ch08-tokio-deep-dive.md) для изучения каналов и примитивов синхронизации, [Глава 12 — Распространенные ошибки](ch12-common-pitfalls.md) для изучения опасностей отмены во время завершения работы

***
