# 16. Основы Async/Await 🔴

> **Что вы узнаете:**
> - Чем модель асинхронности в Rust (`Future`) отличается от горутин в Go и `asyncio` в Python
> - Быстрый старт с Tokio: спавн задач (spawning tasks), `join!` и конфигурация рантайма
> - Распространенные ошибки в асинхронном программировании и способы их исправления
> - Когда нужно переносить блокирующую работу в пул с помощью `spawn_blocking`

## Futures, рантаймы и `async fn`

Асинхронная модель Rust *принципиально отличается* от горутин в Go или библиотеки `asyncio` в Python. Для начала работы достаточно понять три концепции:

1. **`Future` — это ленивый конечный автомат**. Вызов `async fn` ничего не запускает на выполнение, он возвращает `Future` (будущее значение), которое необходимо опрашивать (poll).
2. **Для опроса futures необходим рантайм** (вычислитель/планировщик) — такой как `tokio`, `async-std` или `smol`. Стандартная библиотека описывает типаж `Future`, но не предоставляет встроенного рантайма.
3. **`async fn` — это синтаксический сахар**. Компилятор преобразует такую функцию в конечный автомат, реализующий типаж `Future`.

```rust
// Future — это просто типаж (trait):
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// async fn раскрывается (desugar) в:
// fn fetch_data(url: &str) -> impl Future<Output = Result<Vec<u8>, Error>>
async fn fetch_data(url: &str) -> Result<Vec<u8>, reqwest::Error> {
    let response = reqwest::get(url).await?;  // .await приостанавливает выполнение до готовности
    let bytes = response.bytes().await?;
    Ok(bytes.to_vec())
}
```

### Быстрый старт с Tokio

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust,ignore
use tokio::time::{sleep, Duration};
use tokio::task;

#[tokio::main]
async fn main() {
    // Спавн конкурентных задач (подобно легковесным потокам):
    let handle_a = task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "задача A завершена"
    });

    let handle_b = task::spawn(async {
        sleep(Duration::from_millis(50)).await;
        "задача B завершена"
    });

    // Ожидаем выполнение обеих задач с помощью .await — они работают конкурентно, а не последовательно:
    let (a, b) = tokio::join!(handle_a, handle_b);
    println!("{}, {}", a.unwrap(), b.unwrap());
}
```

### Распространенные ошибки в асинхронном коде

| Ошибка | Почему происходит | Как исправить |
|---------|---------------|-----|
| Блокировки в асинхронном коде | `std::thread::sleep` или вычисления на CPU блокируют исполнитель (executor) | Используйте `tokio::task::spawn_blocking` или `rayon` |
| Ошибки ограничения `Send` | Future, удерживаемая между точками `.await`, содержит тип `!Send` (например, `Rc`, `MutexGuard`) | Измените структуру кода, чтобы сбросить (drop) значения `!Send` перед вызовом `.await` |
| Future не опрашивается | Вызов `async fn` без `.await` или спавна — ничего не происходит | Всегда используйте `.await` или передавайте полученную future в `tokio::spawn` |
| Удержание `MutexGuard` во время `.await` | Ссылочный `std::sync::MutexGuard` является `!Send`; асинхронные задачи могут возобновляться в другом потоке | Используйте `tokio::sync::Mutex` или сбрасывайте гард перед точкой `.await` |
| Случайное последовательное выполнение | Конструкция `let a = foo().await; let b = bar().await;` выполняется последовательно | Используйте `tokio::join!` или `tokio::spawn` для конкурентности |

```rust
// ❌ Блокирование асинхронного исполнителя (executor):
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(5)); // Блокирует весь поток!
}

// ✅ Вынос блокирующей работы в пул:
async fn good() {
    tokio::task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(5)); // Выполняется в пуле блокирующих задач
    }).await.unwrap();
}
```

> **Комплексное руководство по асинхронности**: Подробную информацию о `Stream`, `select!`, безопасности отмены,
> структурированной конкурентности и middleware `tower` см. в нашем специализированном
> руководстве **Async Rust Training**. Этот раздел охватывает лишь необходимый минимум для чтения и
> написания простого асинхронного кода.

### Спавн задач и структурированная конкурентность

Функция `spawn` в Tokio создает новую асинхронную задачу — аналогично `thread::spawn`, но гораздо легче:

```rust,ignore
use tokio::task;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // Спавним три конкурентные задачи
    let h1 = task::spawn(async {
        sleep(Duration::from_millis(200)).await;
        "профиль пользователя получен"
    });

    let h2 = task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "история заказов получена"
    });

    let h3 = task::spawn(async {
        sleep(Duration::from_millis(150)).await;
        "рекомендации получены"
    });

    // Ожидаем завершения всех трех задач конкурентно (не последовательно!)
    let (r1, r2, r3) = tokio::join!(h1, h2, h3);
    println!("{}", r1.unwrap());
    println!("{}", r2.unwrap());
    println!("{}", r3.unwrap());
}
```

**Сравнение `join!`, `try_join!` и `select!`**:

| Макрос | Поведение | Когда использовать |
|-------|----------|----------|
| `join!` | Ожидает завершения ВСЕХ futures | Все задачи должны успешно завершиться |
| `try_join!` | Ожидает все, но прерывается при первой ошибке `Err` | Задачи возвращают тип `Result` |
| `select!` | Возвращает управление, когда завершается ПЕРВАЯ future | Таймауты, отмена операций |

```rust,ignore
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout() -> Result<String, Box<dyn std::error::Error>> {
    let result = timeout(Duration::from_secs(5), async {
        // Симулируем медленный сетевой вызов
        tokio::time::sleep(Duration::from_millis(100)).await;
        Ok::<_, Box<dyn std::error::Error>>("data".to_string())
    }).await??; // Первый ? разворачивает Elapsed, второй ? разворачивает внутренний Result

    Ok(result)
}
```

### Ограничение `Send` и почему Futures должны быть `Send`

Когда вы запускаете future через `tokio::spawn`, её выполнение может возобновиться в другом потоке ОС. Это означает, что будущий результат должен реализовывать `Send`. Типичные ошибки:

```rust,ignore
use std::rc::Rc;

async fn not_send() {
    let rc = Rc::new(42); // Rc реализует !Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", rc); // rc удерживается между точками .await — future является !Send
}

// Решение 1: Сбросить (drop) значение перед .await
async fn fixed_drop() {
    let data = {
        let rc = Rc::new(42);
        *rc // Копируем значение наружу
    }; // rc сбрасывается здесь
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", data); // Просто i32, который реализует Send
}

// Решение 2: Использовать Arc вместо Rc
async fn fixed_arc() {
    let arc = std::sync::Arc::new(42); // Arc реализует Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", arc); // ✅ Future реализует Send
}
```

> **Комплексное руководство по асинхронности**: Подробную информацию о `Stream`, `select!`, безопасности отмены,
> структурированной конкурентности и middleware `tower` см. в нашем специализированном
> руководстве **Async Rust Training**. Этот раздел охватывает лишь необходимый минимум для чтения и
> написания простого асинхронного кода.

> **См. также:** раздел [Глава 5 — Каналы](ch05-channels-and-message-passing.md) для синхронных каналов; [Глава 6 — Конкурентность](ch06-concurrency-vs-parallelism-vs-threads.md) для сравнения потоков ОС и асинхронных задач.

> **Основные выводы — Асинхронное программирование**
> - `async fn` возвращает ленивую `Future` — выполнение не начнется до тех пор, пока вы не примените `.await` или не передадите её планировщику (spawn)
> - Используйте `tokio::task::spawn_blocking` для ресурсоемких (CPU-bound) или блокирующих операций внутри асинхронного контекста
> - Не удерживайте `std::sync::MutexGuard` между вызовами `.await` — используйте вместо него `tokio::sync::Mutex`
> - Передаваемые планировщику futures должны быть `Send` — сбрасывайте (drop) типы `!Send` перед вызовами `.await`

---

### Упражнение: конкурентный загрузчик с таймаутом ★★ (~25 мин)

Напишите асинхронную функцию `fetch_all`, которая запускает три задачи через `tokio::spawn`, каждая из которых симулирует сетевой запрос с помощью `tokio::time::sleep`. Объедините выполнение всех трех задач с помощью `tokio::try_join!`, обернув его в `tokio::time::timeout(Duration::from_secs(5), ...)`. Функция должна возвращать `Result<Vec<String>, ...>` или ошибку, если хотя бы одна задача завершилась неудачей или истек отведенный таймаут.

<details>
<summary>🔑 Решение</summary>

```rust,ignore
use tokio::time::{sleep, timeout, Duration};

async fn fake_fetch(name: &'static str, delay_ms: u64) -> Result<String, String> {
    sleep(Duration::from_millis(delay_ms)).await;
    Ok(format!("{name}: OK"))
}

async fn fetch_all() -> Result<Vec<String>, Box<dyn std::error::Error>> {
    let deadline = Duration::from_secs(5);

    let (a, b, c) = timeout(deadline, async {
        let h1 = tokio::spawn(fake_fetch("svc-a", 100));
        let h2 = tokio::spawn(fake_fetch("svc-b", 200));
        let h3 = tokio::spawn(fake_fetch("svc-c", 150));
        tokio::try_join!(h1, h2, h3)
    })
    .await??;

    Ok(vec![a?, b?, c?])
}

#[tokio::main]
async fn main() {
    let results = fetch_all().await.unwrap();
    for r in &results {
        println!("{r}");
    }
}
```

</details>

***
