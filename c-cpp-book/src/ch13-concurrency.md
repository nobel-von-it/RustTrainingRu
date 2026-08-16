# Многопоточность и конкурентность в Rust

> **Чему вы научитесь:** Модель многопоточности и конкурентности в Rust — потоки, маркерные трейты `Send`/`Sync`, `Mutex<T>`, `Arc<T>`, каналы, а также то, как компилятор предотвращает состояние гонки (data races) на этапе компиляции. Никаких накладных расходов во время выполнения на безопасность потоков, которую вы не используете.

- В Rust есть встроенная поддержка многопоточности, аналогичная `std::thread` в C++.
    - Ключевое отличие: Rust **предотвращает состояние гонки за данные (data races) во время компиляции** с помощью маркерных трейтов `Send` и `Sync`.
    - В C++ совместное использование `std::vector` между потоками без мьютекса приводит к неопределенному поведению (UB), но отлично компилируется. В Rust такой код не скомпилируется.
    - `Mutex<T>` в Rust оборачивает **сами данные**, а не просто доступ к ним — вы буквально не можете прочитать данные без захвата блокировки.
- Функция `thread::spawn()` используется для создания отдельного потока, который выполняет замыкание `||` параллельно.
```rust
use std::thread;
use std::time::Duration;
fn main() {
    let handle = thread::spawn(|| {
        for i in 0..10 {
            println!("Count in thread: {i}!");
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 0..5 {
        println!("Main thread: {i}");
        thread::sleep(Duration::from_millis(5));
    }

    handle.join().unwrap(); // Вызов handle.join() гарантирует, что порожденный поток завершит работу
}
```

# Конкурентность в Rust
- Функция `thread::scope()` может использоваться в тех случаях, когда необходимо заимствовать переменные из окружения. Это работает, так как `thread::scope` ожидает возврата управления из внутреннего потока.
- Попробуйте выполнить это упражнение без `thread::scope`, чтобы увидеть возникающую проблему.
```rust
use std::thread;
fn main() {
  let a = [0, 1, 2];
  thread::scope(|scope| {
      scope.spawn(|| {
          for x in &a {
            println!("{x}");
          }
      });
  });
}
```
----
# Конкурентность в Rust
- Мы также можем использовать ключевое слово `move` для передачи владения потоку. Для типов с трейтом `Copy`, таких как `[i32; 3]`, ключевое слово `move` копирует данные в замыкание, а оригинал остается доступным для дальнейшего использования.
```rust
use std::thread;
fn main() {
  let mut a = [0, 1, 2];
  let handle = thread::spawn(move || {
      for x in a {
        println!("{x}");
      }
  });
  a[0] = 42;    // Не влияет на копию, переданную в поток
  handle.join().unwrap();
}
```

# Конкурентность in Rust
- `Arc<T>` можно использовать для совместного использования ссылок *только для чтения* между несколькими потоками.
    - `Arc` расшифровывается как Atomic Reference Counted (атомарный подсчет ссылок). Объект в памяти освобождается только тогда, когда счетчик ссылок становится равным 0.
    - `Arc::clone()` просто увеличивает счетчик ссылок без клонирования самих данных.
```rust
use std::sync::Arc;
use std::thread;
fn main() {
    let a = Arc::new([0, 1, 2]);
    let mut handles = Vec::new();
    for i in 0..2 {
        let arc = Arc::clone(&a);
        handles.push(thread::spawn(move || {
            println!("Thread: {i} {arc:?}");
        }));
    }
    handles.into_iter().for_each(|h| h.join().unwrap());
}
```

# Конкурентность в Rust
- `Arc<T>` можно комбинировать с `Mutex<T>` для предоставления изменяемых ссылок.
    - `Mutex` защищает лежащие в нем данные и гарантирует, что доступ имеет только тот поток, который удерживает блокировку.
    - Блокировка `MutexGuard` автоматически освобождается, когда она выходит из области видимости (RAII). Примечание: с помощью `std::mem::forget` все еще можно утечь блокировку, поэтому фраза «невозможно забыть разблокировать» точнее, чем «невозможно утечь» (leak).
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
            // Здесь MutexGuard выходит из области видимости — блокировка снимается автоматически
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final count: {}", *counter.lock().unwrap());
    // Вывод: Final count: 5
}
```

# Конкурентность в Rust: RwLock
- `RwLock<T>` разрешает **несколько параллельных читателей** или **одного монопольного писателя** — паттерн блокировки чтения/записи (read/write lock), аналогичный `std::shared_mutex` в C++.
    - Используйте `RwLock`, когда количество операций чтения значительно превышает количество операций записи (например, настройки, кэши).
    - Используйте `Mutex`, когда частота чтения и записи сопоставима или критические секции коротки.
```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("v1.0")));
    let mut handles = Vec::new();

    // Создаем 5 читателей — все они могут работать параллельно
    for i in 0..5 {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let val = config.read().unwrap();  // Несколько читателей работают одновременно
            println!("Reader {i}: {val}");
        }));
    }

    // Один писатель — блокирует работу до тех пор, пока все читатели не закончат
    {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let mut val = config.write().unwrap();  // Эксклюзивный доступ
            *val = String::from("v2.0");
            println!("Writer: updated to {val}");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

# Конкурентность в Rust: отравление мьютексов (Mutex poisoning)
- Если поток **паникует** во время удержания `Mutex` или `RwLock`, блокировка становится **отравленной** (poisoned).
    - Последующие вызовы метода `.lock()` возвращают `Err(PoisonError)`, так как данные могли остаться в несогласованном состоянии.
    - Вы можете восстановить доступ с помощью `.into_inner()`, если уверены, что данные по-прежнему валидны.
    - В C++ нет эквивалента — у `std::mutex` нет концепции отравления; паникующий поток просто оставляет блокировку захваченной (в случае вызова abort/panic).
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let data2 = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut guard = data2.lock().unwrap();
        guard.push(4);
        panic!("oops!");  // Блокировка теперь отравлена
    });

    let _ = handle.join();  // Поток запаниковал

    // Последующие попытки заблокировать возвращают Err(PoisonError)
    match data.lock() {
        Ok(guard) => println!("Data: {guard:?}"),
        Err(poisoned) => {
            println!("Lock was poisoned! Recovering...");
            let guard = poisoned.into_inner();  // Получить доступ к данным в любом случае
            println!("Recovered data: {guard:?}");  // [1, 2, 3, 4] — добавление в вектор (push) произошло до паники
        }
    }
}
```

# Конкурентность в Rust: Атомарные типы (Atomics)
- Для простых счетчиков и флагов типы из модуля `std::sync::atomic` позволяют избежать накладных расходов на `Mutex`.
    - `AtomicBool`, `AtomicI32`, `AtomicU64`, `AtomicUsize` и т.д.
    - Эквивалентно `std::atomic<T>` в C++ — та же модель упорядочения памяти (`Relaxed`, `Acquire`, `Release`, `SeqCst`).
```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", counter.load(Ordering::SeqCst));
    // Вывод: Counter: 10000
}
```

| Примитив | Когда использовать | Эквивалент в C++ |
|-----------|-------------|----------------|
| `Mutex<T>` | Общее изменяемое состояние общего доступа | `std::mutex` + ручное связывание с данными |
| `RwLock<T>` | Нагрузки с преобладанием чтения | `std::shared_mutex` |
| `Atomic*` | Простые счетчики, флаги, неблокирующие паттерны | `std::atomic<T>` |
| `Condvar` | Ожидание выполнения некоторого условия | `std::condition_variable` |

# Конкурентность в Rust: Переменные условий (Condvar)
- `Condvar` (condition variable, переменная условия) позволяет потоку **заснуть до тех пор, пока другой поток не просигнализирует**, что состояние изменилось.
    - Всегда работает в паре с `Mutex`. Паттерн выглядит так: блокировка мьютекса, проверка условия, ожидание в случае неготовности, выполнение действий при готовности.
    - Эквивалентно `std::condition_variable` / `std::condition_variable::wait` в C++.
    - Защищает от **ложных пробуждений** (spurious wakeups) — всегда перепроверяйте условие в цикле (или используйте `wait_while`/`wait_until`).
```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));

    // Создаем рабочий поток (worker), который ждет сигнала
    let pair2 = Arc::clone(&pair);
    let worker = thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut ready = lock.lock().unwrap();
        // wait: засыпает до получения сигнала (всегда перепроверяйте в цикле из-за ложных пробуждений)
        while !*ready {
            ready = cvar.wait(ready).unwrap();
        }
        println!("Worker: condition met, proceeding!");
    });

    // Главный поток выполняет работу, а затем сигнализирует воркеру
    thread::sleep(std::time::Duration::from_millis(100));
    {
        let (lock, cvar) = &*pair;
        let mut ready = lock.lock().unwrap();
        *ready = true;
        cvar.notify_one();  // Будим один ожидающий поток (notify_all() будит все)
    }

    worker.join().unwrap();
}
```

> **Когда использовать Condvar, а когда каналы:** Используйте `Condvar`, когда потоки совместно используют изменяемое состояние и должны ожидать выполнения некоторого условия на этом состоянии (например, «буфер не пуст»). Используйте каналы (`mpsc`), когда потокам нужно обмениваться *сообщениями*. О логике каналов обычно проще рассуждать.

# Конкурентность в Rust
- Каналы в Rust могут использоваться для обмена сообщениями между отправителем (`Sender`) и получателем (`Receiver`).
    - Используется парадигма под названием `mpsc` (Multi-producer, Single-Consumer — множество производителей, один потребитель).
    - Как `send()`, так и `recv()` могут блокировать поток.
```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    tx.send(10).unwrap();
    tx.send(20).unwrap();
    
    println!("Received: {:?}", rx.recv());
    println!("Received: {:?}", rx.recv());

    let tx2 = tx.clone();
    tx2.send(30).unwrap();
    println!("Received: {:?}", rx.recv());
}
```

# Конкурентность в Rust
- Каналы можно сочетать с потоками
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    for _ in 0..2 {
        let tx2 = tx.clone();
        thread::spawn(move || {
            let thread_id = thread::current().id();
            for i in 0..10 {
                tx2.send(format!("Message {i}")).unwrap();
                println!("{thread_id:?}: sent Message {i}");
            }
            println!("{thread_id:?}: done");
        });
    }

    // Удаляем оригинальный отправитель, чтобы rx.iter() завершился, когда все клонированные отправители будут уничтожены
    drop(tx);

    thread::sleep(Duration::from_millis(100));

    for msg in rx.iter() {
        println!("Main: got {msg}");
    }
}
```



## Почему Rust предотвращает гонки за данные: Send и Sync

- Rust использует два маркерных трейта для обеспечения потокобезопасности на этапе компиляции:
    - `Send`: Тип является `Send`, если его значение можно безопасно **передать** в другой поток.
    - `Sync`: Тип является `Sync`, если им можно безопасно **делиться** (через ссылку `&T`) между потоками.
- Большинство типов автоматически реализуют `Send + Sync`. Известные исключения:
    - `Rc<T>` не является ни Send, ни Sync (используйте `Arc<T>` для потоков).
    - `Cell<T>` и `RefCell<T>` не являются `Sync` (используйте `Mutex<T>` или `RwLock<T>`).
    - Сырые указатели (`*const T`, `*mut T`) не являются ни Send, ни Sync.
- Именно поэтому компилятор не позволяет вам использовать `Rc<T>` между потоками — он просто не реализует трейт `Send`.
- `Arc<Mutex<T>>` — это потокобезопасный эквивалент для `Rc<RefCell<T>>`.

> **Интуитивное понимание (от Джона Дженгсета):** Представьте значения в виде игрушек.
> **`Send`** = вы можете **отдать свою игрушку** другому ребенку (потоку) — передача владения безопасна.
> **`Sync`** = вы можете **разрешить другим играть с вашей игрушкой** одновременно — совместное использование по ссылке безопасно.
> Тип `Rc<T>` имеет хрупкий (неатомарный) счетчик ссылок; его передача или совместное использование приведет к повреждению счетчика, поэтому он не является ни `Send`, ни `Sync`.


# Упражнение: Многопоточный подсчет слов

🔴 **Сложная задача** — сочетает в себе потоки, Arc, Mutex и HashMap

- Дана коллекция `Vec<String>` текстовых строк. Породите по одному потоку на каждую строку для подсчета слов в ней.
- Используйте `Arc<Mutex<HashMap<String, usize>>>` для сбора результатов.
- Выведите общее количество слов по всем строкам.
- **Бонус**: Попробуйте реализовать это с помощью каналов (`mpsc`) вместо разделяемого состояния.

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let lines = vec![
        "the quick brown fox".to_string(),
        "jumps over the lazy dog".to_string(),
        "the fox is quick".to_string(),
    ];

    let word_counts: Arc<Mutex<HashMap<String, usize>>> =
        Arc::new(Mutex::new(HashMap::new()));

    let mut handles = vec![];
    for line in &lines {
        let line = line.clone();
        let counts = Arc::clone(&word_counts);
        handles.push(thread::spawn(move || {
            for word in line.split_whitespace() {
                let mut map = counts.lock().unwrap();
                *map.entry(word.to_lowercase()).or_insert(0) += 1;
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let counts = word_counts.lock().unwrap();
    let total: usize = counts.values().sum();
    println!("Word frequencies: {counts:#?}");
    println!("Total words: {total}");
}
// Вывод (порядок может отличаться):
// Word frequencies: {
//     "the": 3,
//     "quick": 2,
//     "brown": 1,
//     "fox": 2,
//     "jumps": 1,
//     "over": 1,
//     "lazy": 1,
//     "dog": 1,
//     "is": 1,
// }
// Total words: 13
```

</details>
