# 9. Когда Tokio не подходит 🟡

> **Чему вы научитесь:**
> - Проблеме `'static`: когда `tokio::spawn` заставляет вас использовать `Arc` повсюду
> - Использованию `LocalSet` для `!Send`-фьючеров
> - Использованию `FuturesUnordered` для конкурентности с заимствованиями (без вызова `spawn`)
> - Использованию `JoinSet` для управляемых групп задач
> - Написанию библиотек, независимых от среды выполнения (runtime-agnostic)

```mermaid
graph TD
    START["Нужны конкурентные фьючеры?"] --> STATIC{"Могут ли фьючеры быть 'static?"}
    STATIC -->|Да| SEND{"Фьючеры реализуют Send?"}
    STATIC -->|Нет| FU["FuturesUnordered<br/>Выполняются на текущей задаче"]
    SEND -->|Да| SPAWN["tokio::spawn<br/>Многопоточность"]
    SEND -->|Нет| LOCAL["LocalSet<br/>Однопоточность"]
    SPAWN --> MANAGE{"Нужно отслеживать/отменять задачи?"}
    MANAGE -->|Да| JOINSET["JoinSet / TaskTracker"]
    MANAGE -->|Нет| HANDLE["JoinHandle"]

    style START fill:#f5f5f5,stroke:#333,color:#000
    style FU fill:#d4efdf,stroke:#27ae60,color:#000
    style SPAWN fill:#e8f4f8,stroke:#2980b9,color:#000
    style LOCAL fill:#fef9e7,stroke:#f39c12,color:#000
    style JOINSET fill:#e8daef,stroke:#8e44ad,color:#000
    style HANDLE fill:#e8f4f8,stroke:#2980b9,color:#000
```

## Проблема фьючеров 'static

Метод `spawn` в Tokio требует `'static`-фьючеров. Это означает, что вы не можете заимствовать локальные данные в запускаемых задачах:

```rust
async fn process_items(items: &[String]) {
    // ❌ Так сделать нельзя — items заимствуется, а не является 'static
    // for item in items {
    //     tokio::spawn(async {
    //         process(item).await;
    //     });
    // }

    // 😐 Обходной путь 1: Клонировать всё
    for item in items {
        let item = item.clone();
        tokio::spawn(async move {
            process(&item).await;
        });
    }

    // 😐 Обходной путь 2: Использовать Arc
    let items = Arc::new(items.to_vec());
    for i in 0..items.len() {
        let items = Arc::clone(&items);
        tokio::spawn(async move {
            process(&items[i]).await;
        });
    }
}
```

Это может раздражать! В Go вы можете просто написать `go func() { use(item) }` с замыканием. В Rust же система владения заставляет вас думать о том, кто владеет ресурсом и как долго он живет.

### Альтернативы `tokio::spawn`

Не каждая задача требует вызова `spawn`. Вот три инструмента, каждый из которых решает *своё* ограничение:

```rust
// 1. FuturesUnordered — полностью избегает 'static (без spawn!)
use futures::stream::{FuturesUnordered, StreamExt};

async fn process_items(items: &[String]) {
    let futures: FuturesUnordered<_> = items
        .iter()
        .map(|item| async move {
            // ✅ Можно заимствовать item — нет spawn, 'static не требуется!
            process(item).await
        })
        .collect();

    // Доводим все фьючеры до завершения
    futures.for_each(|result| async move {
        println!("Результат: {result:?}");
    }).await;
}

// 2. tokio::task::LocalSet — запуск !Send-фьючеров на текущем потоке
//    ⚠️ Все еще требует 'static — решает проблему с Send, а не со 'static
use tokio::task::LocalSet;

let local_set = LocalSet::new();
local_set.run_until(async {
    tokio::task::spawn_local(async {
        // Можно использовать Rc, Cell и другие !Send типы здесь
        let rc = std::rc::Rc::new(42);
        println!("{rc}");
    }).await.unwrap();
}).await;

// 3. tokio JoinSet (tokio 1.21+) — управляемый набор запущенных задач
//    ⚠️ Все еще требует 'static + Send — решает задачу управления задачами,
//    а не проблему 'static. Полезен для отслеживания, отмены и ожидания
//    завершения динамической группы задач.
use tokio::task::JoinSet;

async fn with_joinset() {
    let mut set = JoinSet::new();

    for i in 0..10 {
        // i реализует Copy и перемещается в замыкание — уже является 'static.
        // Вам все еще понадобится Arc или клонирование для заимствованных данных.
        set.spawn(async move {
            tokio::time::sleep(Duration::from_millis(100)).await;
            i * 2
        });
    }

    while let Some(result) = set.join_next().await {
        println!("Задача завершена: {:?}", result.unwrap());
    }
}
```

> **Какой инструмент какую задачу решает?**
>
> | Ограничение, с которым вы столкнулись | Инструмент | Избегает `'static`? | Избегает `Send`? |
> |---|---|---|---|
> | Невозможно сделать фьючеры `'static` | `FuturesUnordered` | ✅ Да | ✅ Да |
> | Фьючеры являются `'static`, но `!Send` | `LocalSet` | ❌ Нет | ✅ Да |
> | Нужно отслеживать / отменять запущенные задачи | `JoinSet` | ❌ Нет | ❌ Нет |

### Легковесные среды выполнения для библиотек

Если вы пишете библиотеку — не заставляйте пользователей использовать Tokio:

```rust
// ❌ ПЛОХО: Библиотека навязывает tokio пользователям
pub async fn my_lib_function() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    // Теперь ваши пользователи ОБЯЗАНЫ использовать tokio
}

// ✅ ХОРОШО: Библиотека независима от рантайма
pub async fn my_lib_function() {
    // Используйте только типы из std::future и futures-crate
    do_computation().await;
}

// ✅ ХОРОШО: Принимаем обобщенный фьючер для операций ввода-вывода
pub async fn fetch_with_retry<F, Fut, T, E>(
    operation: F,
    max_retries: usize,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    for attempt in 0..max_retries {
        match operation().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt == max_retries - 1 => return Err(e),
            Err(_) => continue,
        }
    }
    unreachable!()
}
```

> **Правило**: Библиотеки должны зависеть от библиотеки `futures`, а не от `tokio`.
> Приложения должны зависеть от `tokio` (или выбранного ими рантайма).
> Это сохраняет модульность экосистемы.

<details>
<summary><strong>🏋️ Упражнение: FuturesUnordered против Spawn</strong> (нажмите для раскрытия)</summary>

**Задача**: Напишите одну и ту же функцию двумя способами — один раз с помощью `tokio::spawn` (требует `'static`), а второй раз с использованием `FuturesUnordered` (заимствует данные). Функция принимает `&[String]` и возвращает длину каждой строки после имитации асинхронного поиска.

Сравните: какой подход требует вызова `.clone()`? Какой из них может просто заимствовать входной срез?

<details>
<summary>🔑 Решение</summary>

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use tokio::time::{sleep, Duration};

// Версия 1: tokio::spawn — требует 'static, нужно клонировать
async fn lengths_with_spawn(items: &[String]) -> Vec<usize> {
    let mut handles = Vec::new();
    for item in items {
        let owned = item.clone(); // Нужно клонировать — spawn требует 'static
        handles.push(tokio::spawn(async move {
            sleep(Duration::from_millis(10)).await;
            owned.len()
        }));
    }

    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// Версия 2: FuturesUnordered — заимствует данные, клонирование не требуется
async fn lengths_without_spawn(items: &[String]) -> Vec<usize> {
    let futures: FuturesUnordered<_> = items
        .iter()
        .map(|item| async move {
            sleep(Duration::from_millis(10)).await;
            item.len() // ✅ Заимствует item — без клонирования!
        })
        .collect();

    futures.collect().await
}

#[tokio::test]
async fn test_both_versions() {
    let items = vec!["hello".into(), "world".into(), "rust".into()];

    let v1 = lengths_with_spawn(&items).await;
    // Примечание: v1 сохраняет исходный порядок (последовательный join)

    let mut v2 = lengths_without_spawn(&items).await;
    v2.sort(); // FuturesUnordered возвращает результаты в порядке их завершения

    assert_eq!(v1, vec![5, 5, 4]);
    assert_eq!(v2, vec![4, 5, 5]);
}
```

**Ключевой вывод**: `FuturesUnordered` позволяет обойти требование `'static`, выполняя все фьючеры на текущей задаче (без миграции потоков). Компромисс: все фьючеры делят одну общую задачу — если один из них блокирует поток, все остальные замирают. Используйте `spawn` для задач с интенсивными вычислениями на CPU, которые должны выполняться на отдельных потоках.

</details>
</details>

> **Основные выводы — Когда Tokio не подходит**
> - `FuturesUnordered` выполняет фьючеры конкурентно в рамках текущей задачи — требование `'static` отсутствует
> - `LocalSet` позволяет запускать `!Send`-фьючеры на однопоточном исполнителе
> - `JoinSet` (tokio 1.21+) предоставляет управляемые группы задач с автоматической очисткой
> - Для библиотек: зависьте только от `std::future::Future` + библиотеки `futures`, а не от `tokio` напрямую

> **См. также:** [Глава 8 — Глубокое погружение в Tokio](ch08-tokio-deep-dive.md) для понимания того, когда spawn является правильным выбором, и [Глава 11 — Streams и AsyncIterator](ch11-streams-and-asynciterator.md) для знакомства с методом `buffer_unordered()` как еще одним ограничителем конкурентности.

***
