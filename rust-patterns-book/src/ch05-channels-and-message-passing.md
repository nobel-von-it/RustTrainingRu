# 5. Каналы и передача сообщений 🟢

> **Чему вы научитесь:**
> - Основам `std::sync::mpsc` и тому, когда стоит перейти на crossbeam-channel
> - Выбору каналов с помощью макроса `select!` для обработки сообщений из нескольких источников
> - Буферизованным (bounded) и небуферизованным (unbounded) каналам и стратегиям обратного давления (backpressure)
> - Шаблону актеров (actor pattern) для инкапсуляции конкурентного состояния

## std::sync::mpsc — стандартный канал

Стандартная библиотека Rust предоставляет канал с несколькими производителями и одним потребителем (multi-producer, single-consumer, MPSC):

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // Создаем канал: tx (передатчик) и rx (приемник)
    let (tx, rx) = mpsc::channel();

    // Запускаем поток-производитель (producer)
    let tx1 = tx.clone(); // Клонируем для поддержки нескольких производителей
    thread::spawn(move || {
        for i in 0..5 {
            tx1.send(format!("producer-1: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // Второй производитель
    thread::spawn(move || {
        for i in 0..5 {
            tx.send(format!("producer-2: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(150));
        }
    });

    // Потребитель: получаем все сообщения
    for msg in rx {
        // Итератор rx завершается, когда ВСЕ отправители (tx) удалены (dropped)
        println!("Получено: {msg}");
    }
    println!("Все производители завершили работу.");
}
```

> **Примечание**: Вызов `.unwrap()` на методе `.send()` используется для краткости. Он вызывает панику, если приемник уже был удален. В промышленном (production) коде необходимо обрабатывать ошибку `SendError` должным образом.

**Ключевые свойства**:
- **Небуферизованный (unbounded)** по умолчанию (может переполнить память, если потребитель работает медленно)
- `mpsc::sync_channel(N)` создает **буферизованный (bounded)** канал с механизмом обратного давления (backpressure)
- `rx.recv()` блокирует текущий поток до тех пор, пока не поступит сообщение
- `rx.try_recv()` возвращает управление немедленно с ошибкой `Err(TryRecvError::Empty)`, если в канале ничего нет
- Канал закрывается, когда все отправители (`Sender`) удалены

```rust
// Буферизованный канал с обратным давлением:
let (tx, rx) = mpsc::sync_channel(10); // Буфер на 10 сообщений

thread::spawn(move || {
    for i in 0..1000 {
        tx.send(i).unwrap(); // БЛОКИРУЕТ поток, если буфер полон — естественное обратное давление (backpressure)
    }
});
```

> **Примечание**: `.unwrap()` используется для краткости. В реальном коде обрабатывайте `SendError` (когда приемник удален) вместо вызова паники.

### crossbeam-channel — рабочая лошадка для Production

Библиотека `crossbeam-channel` является стандартом де-факто для использования каналов в реальных проектах. Она быстрее, чем `std::sync::mpsc`, и поддерживает архитектуру с несколькими потребителями (multi-producer, multi-consumer, MPMC):

```rust,ignore
// Cargo.toml:
//   [dependencies]
//   crossbeam-channel = "0.5"
use crossbeam_channel::{bounded, unbounded, select, Sender, Receiver};
use std::thread;
use std::time::Duration;

fn main() {
    // Буферизованный MPMC-канал
    let (tx, rx) = bounded::<String>(100);

    // Несколько производителей
    for id in 0..4 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..10 {
                tx.send(format!("worker-{id}: item-{i}")).unwrap();
            }
        });
    }
    drop(tx); // Удаляем исходный отправитель, чтобы канал мог закрыться

    // Несколько потребителей (это невозможно сделать с std::sync::mpsc!)
    let rx2 = rx.clone();
    let consumer1 = thread::spawn(move || {
        while let Ok(msg) = rx.recv() {
            println!("[consumer-1] {msg}");
        }
    });
    let consumer2 = thread::spawn(move || {
        while let Ok(msg) = rx2.recv() {
            println!("[consumer-2] {msg}");
        }
    });

    consumer1.join().unwrap();
    consumer2.join().unwrap();
}
```

### Выбор каналов (select!)

Прослушивание нескольких каналов одновременно — аналогично оператору `select` в Go:

```rust,ignore
use crossbeam_channel::{bounded, tick, after, select};
use std::time::Duration;

fn main() {
    let (work_tx, work_rx) = bounded::<String>(10);
    let ticker = tick(Duration::from_secs(1));        // Периодический тик
    let deadline = after(Duration::from_secs(10));     // Однократный таймаут

    // Производитель
    let tx = work_tx.clone();
    std::thread::spawn(move || {
        for i in 0..100 {
            tx.send(format!("job-{i}")).unwrap();
            std::thread::sleep(Duration::from_millis(500));
        }
    });
    drop(work_tx);

    loop {
        select! {
            recv(work_rx) -> msg => {
                match msg {
                    Ok(job) => println!("Обработка: {job}"),
                    Err(_) => {
                        println!("Канал работы закрыт");
                        break;
                    }
                }
            },
            recv(ticker) -> _ => {
                println!("Тик — пульс (heartbeat)");
            },
            recv(deadline) -> _ => {
                println!("Предельный срок (deadline) достигнут — завершение работы");
                break;
            },
        }
    }
}
```

> **Сравнение с Go**: Это работает точно так же, как оператор `select` в Go для каналов.
> Макрос `select!` из библиотеки crossbeam рандомизирует порядок опроса для предотвращения голодания потоков (starvation), как и в Go.

### Буферизованные и небуферизованные каналы и обратное давление (Backpressure)

| Тип | Поведение при заполнении | Память | Сценарий использования |
|------|-------------------|--------|----------|
| **Небуферизованный (Unbounded)** | Никогда не блокирует поток (куча разрастается) | Не ограничена ⚠️ | Редко — только когда производитель гарантированно медленнее потребителя |
| **Буферизованный (Bounded)** | `send()` блокирует поток, пока не появится место | Фиксированная | По умолчанию в реальных проектах — предотвращает ошибки нехватки памяти (OOM) |
| **Рандеву (Rendezvous)** (bounded(0)) | `send()` блокирует поток до тех пор, пока приемник не будет готов принять данные | Отсутствует | Синхронизация / прямая передача из рук в руки |

```rust
// Канал рандеву (Rendezvous) — нулевая емкость, прямая передача
let (tx, rx) = crossbeam_channel::bounded(0);
// tx.send(x) блокирует поток до тех пор, пока не будет вызван rx.recv(), и наоборот.
// Это позволяет точно синхронизировать два потока.
```

**Правило**: Всегда используйте буферизованные (bounded) каналы в реальных проектах, если только вы не можете доказать, что производитель никогда не обгонит потребителя.

### Шаблон актеров (Actor Pattern) с использованием каналов

Шаблон актеров использует каналы для сериализации доступа к изменяемому состоянию — мьютексы не требуются:

```rust
use std::sync::mpsc;
use std::thread;

// Сообщения, которые может принимать актер
enum CounterMsg {
    Increment,
    Decrement,
    Get(mpsc::Sender<i64>), // Канал для ответа
}

struct CounterActor {
    count: i64,
    rx: mpsc::Receiver<CounterMsg>,
}

impl CounterActor {
    fn new(rx: mpsc::Receiver<CounterMsg>) -> Self {
        CounterActor { count: 0, rx }
    }

    fn run(mut self) {
        while let Ok(msg) = self.rx.recv() {
            match msg {
                CounterMsg::Increment => self.count += 1,
                CounterMsg::Decrement => self.count -= 1,
                CounterMsg::Get(reply) => {
                    let _ = reply.send(self.count);
                }
            }
        }
    }
}

// Дескриптор актера — дешево клонируется, реализует Send + Sync
#[derive(Clone)]
struct Counter {
    tx: mpsc::Sender<CounterMsg>,
}

impl Counter {
    fn spawn() -> Self {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || CounterActor::new(rx).run());
        Counter { tx }
    }

    fn increment(&self) { let _ = self.tx.send(CounterMsg::Increment); }
    fn decrement(&self) { let _ = self.tx.send(CounterMsg::Decrement); }

    fn get(&self) -> i64 {
        let (reply_tx, reply_rx) = mpsc::channel();
        self.tx.send(CounterMsg::Get(reply_tx)).unwrap();
        reply_rx.recv().unwrap()
    }
}

fn main() {
    let counter = Counter::spawn();

    // Несколько потоков могут безопасно использовать счетчик — без мьютексов!
    let handles: Vec<_> = (0..10).map(|_| {
        let counter = counter.clone();
        thread::spawn(move || {
            for _ in 0..1000 {
                counter.increment();
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
    println!("Итоговое значение: {}", counter.get()); // 10000
}
```

> **Когда использовать актеров, а когда мьютексы**: Актеры отлично подходят, когда состояние имеет сложные инварианты, операции занимают много времени или вы хотите упорядочить доступ к данным, не задумываясь о порядке блокировок. Мьютексы проще для коротких критических секций.

> **Основные выводы — Каналы**
> - `crossbeam-channel` — рабочая лошадка для продакшена — она быстрее и богаче возможностями, чем `std::sync::mpsc`
> - Макрос `select!` заменяет сложный опрос нескольких источников декларативным выбором каналов
> - Буферизованные каналы обеспечивают естественное обратное давление (backpressure); небуферизованные каналы несут в себе риск нехватки памяти (OOM)

> **См. также:** [Гл. 6 — Конкурентность](ch06-concurrency-vs-parallelism-vs-threads.md) для изучения потоков, мьютексов и общего состояния. [Гл. 15 — Асинхронность](ch16-asyncawait-essentials.md) для изучения асинхронных каналов (`tokio::sync::mpsc`).

---

### Упражнение: Пул воркеров на каналах ★★★ (~45 мин)

Создайте пул рабочих потоков (воркеров) с использованием каналов, где:
- Диспетчер отправляет структуры `Job` через канал
- N воркеров принимают задачи и отправляют результаты обратно
- Используется `std::sync::mpsc` совместно с `Arc<Mutex<Receiver>>` для организации общей очереди задач

<details>
<summary>🔑 Решение</summary>

```rust
use std::sync::mpsc;
use std::thread;

struct Job {
    id: u64,
    data: String,
}

struct JobResult {
    job_id: u64,
    output: String,
    worker_id: usize,
}

fn worker_pool(jobs: Vec<Job>, num_workers: usize) -> Vec<JobResult> {
    let (job_tx, job_rx) = mpsc::channel::<Job>();
    let (result_tx, result_rx) = mpsc::channel::<JobResult>();

    let job_rx = std::sync::Arc::new(std::sync::Mutex::new(job_rx));

    let mut handles = Vec::new();
    for worker_id in 0..num_workers {
        let job_rx = job_rx.clone();
        let result_tx = result_tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                let job = {
                    let rx = job_rx.lock().unwrap();
                    rx.recv()
                };
                match job {
                    Ok(job) => {
                        let output = format!("обработано '{}' воркером {worker_id}", job.data);
                        result_tx.send(JobResult {
                            job_id: job.id, output, worker_id,
                        }).unwrap();
                    }
                    Err(_) => break,
                }
            }
        }));
    }
    drop(result_tx);

    let num_jobs = jobs.len();
    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx);

    let results: Vec<_> = result_rx.into_iter().collect();
    assert_eq!(results.len(), num_jobs);

    for h in handles { h.join().unwrap(); }
    results
}

fn main() {
    let jobs: Vec<Job> = (0..20).map(|i| Job {
        id: i, data: format!("задача-{i}"),
    }).collect();

    let results = worker_pool(jobs, 4);
    for r in &results {
        println!("[воркер {}] задача {}: {}", r.worker_id, r.job_id, r.output);
    }
}
```

</details>

***
