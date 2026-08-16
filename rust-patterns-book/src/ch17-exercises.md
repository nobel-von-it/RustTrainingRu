## Упражнения

### Упражнение 1: типобезопасный конечный автомат ★★ (~30 мин)

Создайте конечный автомат светофора с помощью паттерна состояний типов (type-state). Светофор должен переключаться в строгом порядке `Red → Green → Yellow → Red`, и никакие другие переходы не должны быть возможны.

<details>
<summary>🔑 Решение</summary>

```rust
use std::marker::PhantomData;

struct Red;
struct Green;
struct Yellow;

struct TrafficLight<State> {
    _state: PhantomData<State>,
}

impl TrafficLight<Red> {
    fn new() -> Self {
        println!("🔴 Red — STOP");
        TrafficLight { _state: PhantomData }
    }

    fn go(self) -> TrafficLight<Green> {
        println!("🟢 Green — GO");
        TrafficLight { _state: PhantomData }
    }
}

impl TrafficLight<Green> {
    fn caution(self) -> TrafficLight<Yellow> {
        println!("🟡 Yellow — CAUTION");
        TrafficLight { _state: PhantomData }
    }
}

impl TrafficLight<Yellow> {
    fn stop(self) -> TrafficLight<Red> {
        println!("🔴 Red — STOP");
        TrafficLight { _state: PhantomData }
    }
}

fn main() {
    let light = TrafficLight::new(); // Red
    let light = light.go();          // Green
    let light = light.caution();     // Yellow
    let light = light.stop();        // Red

    // light.caution(); // ❌ Ошибка компиляции: нет метода `caution` для Red
    // TrafficLight::new().stop(); // ❌ Ошибка компиляции: нет метода `stop` для Red
}
```

**Основной вывод**: невалидные переходы вызывают ошибки компиляции, а не панику во время выполнения.

</details>

---

### Упражнение 2: размерности величин с помощью PhantomData ★★ (~30 мин)

Расширьте паттерн размерностей величин (unit-of-measure) из главы 4 (Ch4), чтобы поддерживать:
- `Meters`, `Seconds`, `Kilograms`
- Сложение величин одинаковой размерности
- Умножение: `Meters * Meters = SquareMeters`
- Деление: `Meters / Seconds = MetersPerSecond`

<details>
<summary>🔑 Решение</summary>

```rust
use std::marker::PhantomData;
use std::ops::{Add, Mul, Div};

#[derive(Clone, Copy)]
struct Meters;
#[derive(Clone, Copy)]
struct Seconds;
#[derive(Clone, Copy)]
struct Kilograms;
#[derive(Clone, Copy)]
struct SquareMeters;
#[derive(Clone, Copy)]
struct MetersPerSecond;

#[derive(Debug, Clone, Copy)]
struct Qty<U> {
    value: f64,
    _unit: PhantomData<U>,
}

impl<U> Qty<U> {
    fn new(v: f64) -> Self { Qty { value: v, _unit: PhantomData } }
}

impl<U> Add for Qty<U> {
    type Output = Qty<U>;
    fn add(self, rhs: Self) -> Self::Output { Qty::new(self.value + rhs.value) }
}

impl Mul<Qty<Meters>> for Qty<Meters> {
    type Output = Qty<SquareMeters>;
    fn mul(self, rhs: Qty<Meters>) -> Qty<SquareMeters> {
        Qty::new(self.value * rhs.value)
    }
}

impl Div<Qty<Seconds>> for Qty<Meters> {
    type Output = Qty<MetersPerSecond>;
    fn div(self, rhs: Qty<Seconds>) -> Qty<MetersPerSecond> {
        Qty::new(self.value / rhs.value)
    }
}

fn main() {
    let width = Qty::<Meters>::new(5.0);
    let height = Qty::<Meters>::new(3.0);
    let area = width * height; // Qty<SquareMeters>
    println!("Area: {:.1} m²", area.value);

    let dist = Qty::<Meters>::new(100.0);
    let time = Qty::<Seconds>::new(9.58);
    let speed = dist / time;
    println!("Speed: {:.2} m/s", speed.value);

    let sum = width + height; // Одинаковые размерности ✅
    println!("Sum: {:.1} m", sum.value);

    // let bad = width + time; // ❌ Ошибка компиляции: нельзя сложить Meters и Seconds
}
```

</details>

---

### Упражнение 3: пул воркеров на основе каналов ★★★ (~45 мин)

Создайте пул воркеров (worker pool) с использованием каналов, в котором:
- Диспетчер отправляет структуры `Job` через канал
- N воркеров считывают задачи и отправляют результаты обратно
- Используйте `crossbeam-channel` (или `std::sync::mpsc`, если библиотека crossbeam недоступна)

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

    // Оборачиваем получатель (receiver) в Arc<Mutex> для совместного использования воркерами
    let job_rx = std::sync::Arc::new(std::sync::Mutex::new(job_rx));

    // Спавним воркеры
    let mut handles = Vec::new();
    for worker_id in 0..num_workers {
        let job_rx = job_rx.clone();
        let result_tx = result_tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                // Блокировка, получение, разблокировка — короткая критическая секция
                let job = {
                    let rx = job_rx.lock().unwrap();
                    rx.recv() // Блокирует поток до появления задачи или закрытия канала
                };
                match job {
                    Ok(job) => {
                        let output = format!("обработано '{}' воркером {worker_id}", job.data);
                        result_tx.send(JobResult {
                            job_id: job.id,
                            output,
                            worker_id,
                        }).unwrap();
                    }
                    Err(_) => break, // Канал закрыт — выходим из цикла
                }
            }
        }));
    }
    drop(result_tx); // Сбрасываем нашу копию, чтобы канал результатов закрылся, когда воркеры закончат работу

    // Распределяем задачи
    let num_jobs = jobs.len();
    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx); // Закрываем канал задач — воркеры завершат работу после обработки оставшихся задач

    // Собираем результаты
    let mut results = Vec::new();
    for result in result_rx {
        results.push(result);
    }
    assert_eq!(results.len(), num_jobs);

    for h in handles { h.join().unwrap(); }
    results
}

fn main() {
    let jobs: Vec<Job> = (0..20).map(|i| Job {
        id: i,
        data: format!("task-{i}"),
    }).collect();

    let results = worker_pool(jobs, 4);
    for r in &results {
        println!("[worker {}] job {}: {}", r.worker_id, r.job_id, r.output);
    }
}
```

</details>

---

### Упражнение 4: конвейер комбинаторов высшего порядка ★★ (~25 мин)

Создайте структуру `Pipeline`, которая объединяет преобразования в цепочку. Она должна поддерживать метод `.pipe(f)` для добавления преобразования и `.execute(input)` для запуска всей цепочки.

<details>
<summary>🔑 Решение</summary>

```rust
struct Pipeline<T> {
    transforms: Vec<Box<dyn Fn(T) -> T>>,
}

impl<T: 'static> Pipeline<T> {
    fn new() -> Self {
        Pipeline { transforms: Vec::new() }
    }

    fn pipe(mut self, f: impl Fn(T) -> T + 'static) -> Self {
        self.transforms.push(Box::new(f));
        self
    }

    fn execute(self, input: T) -> T {
        self.transforms.into_iter().fold(input, |val, f| f(val))
    }
}

fn main() {
    let result = Pipeline::new()
        .pipe(|s: String| s.trim().to_string())
        .pipe(|s| s.to_uppercase())
        .pipe(|s| format!(">>> {s} <<<"))
        .execute("  hello world  ".to_string());

    println!("{result}"); // >>> HELLO WORLD <<<

    // Числовой конвейер:
    let result = Pipeline::new()
        .pipe(|x: i32| x * 2)
        .pipe(|x| x + 10)
        .pipe(|x| x * x)
        .execute(5);

    println!("{result}"); // (5*2 + 10)^2 = 400
}
```

**Бонус**: обобщенный конвейер, который изменяет тип данных между этапами, требовал бы другой конструкции — в ней каждый вызов `.pipe()` возвращал бы `Pipeline` с новым типом выходных данных (это требует более сложной работы с обобщениями).

</details>

---

### Упражнение 5: иерархия ошибок с помощью thiserror ★★ (~30 мин)

Спроектируйте иерархию типов ошибок для приложения обработки файлов, в котором возможны сбои при вводе-выводе (I/O), парсинге (JSON и CSV) и валидации. Используйте библиотеку `thiserror` и продемонстрируйте распространение ошибок с помощью оператора `?`.

<details>
<summary>🔑 Решение</summary>

```rust,ignore
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("ошибка ввода-вывода (I/O): {0}")]
    Io(#[from] std::io::Error),

    #[error("ошибка парсинга JSON: {0}")]
    Json(#[from] serde_json::Error),

    #[error("ошибка CSV на строке {line}: {message}")]
    Csv { line: usize, message: String },

    #[error("ошибка валидации: {field} — {reason}")]
    Validation { field: String, reason: String },
}

fn read_file(path: &str) -> Result<String, AppError> {
    Ok(std::fs::read_to_string(path)?) // io::Error → AppError::Io через #[from]
}

fn parse_json(content: &str) -> Result<serde_json::Value, AppError> {
    Ok(serde_json::from_str(content)?) // serde_json::Error → AppError::Json
}

fn validate_name(value: &serde_json::Value) -> Result<String, AppError> {
    let name = value.get("name")
        .and_then(|v| v.as_str())
        .ok_or_else(|| AppError::Validation {
            field: "name".into(),
            reason: "должно быть ненулевой строкой".into(),
        })?;

    if name.is_empty() {
        return Err(AppError::Validation {
            field: "name".into(),
            reason: "не должно быть пустым".into(),
        });
    }

    Ok(name.to_string())
}

fn process_file(path: &str) -> Result<String, AppError> {
    let content = read_file(path)?;
    let json = parse_json(&content)?;
    let name = validate_name(&json)?;
    Ok(name)
}

fn main() {
    match process_file("config.json") {
        Ok(name) => println!("Name: {name}"),
        Err(e) => eprintln!("Error: {e}"),
    }
}
```

</details>

---

### Упражнение 6: обобщенный типаж с ассоциированными типами ★★★ (~40 мин)

Спроектируйте типаж `Repository<T>` с ассоциированными типами `Error` и `Id`. Реализуйте его для хранилища в оперативной памяти (in-memory store) и продемонстрируйте типобезопасность на этапе компиляции.

<details>
<summary>🔑 Решение</summary>

```rust
use std::collections::HashMap;

trait Repository {
    type Item;
    type Id;
    type Error;

    fn get(&self, id: &Self::Id) -> Result<Option<&Self::Item>, Self::Error>;
    fn insert(&mut self, item: Self::Item) -> Result<Self::Id, Self::Error>;
    fn delete(&mut self, id: &Self::Id) -> Result<bool, Self::Error>;
}

#[derive(Debug, Clone)]
struct User {
    name: String,
    email: String,
}

struct InMemoryUserRepo {
    data: HashMap<u64, User>,
    next_id: u64,
}

impl InMemoryUserRepo {
    fn new() -> Self {
        InMemoryUserRepo { data: HashMap::new(), next_id: 1 }
    }
}

// Тип ошибки — Infallible (операции в памяти никогда не завершаются ошибкой)
impl Repository for InMemoryUserRepo {
    type Item = User;
    type Id = u64;
    type Error = std::convert::Infallible;

    fn get(&self, id: &u64) -> Result<Option<&User>, Self::Error> {
        Ok(self.data.get(id))
    }

    fn insert(&mut self, item: User) -> Result<u64, Self::Error> {
        let id = self.next_id;
        self.next_id += 1;
        self.data.insert(id, item);
        Ok(id)
    }

    fn delete(&mut self, id: &u64) -> Result<bool, Self::Error> {
        Ok(self.data.remove(id).is_some())
    }
}

// Обобщенная функция работает с ЛЮБЫМ репозиторием:
fn create_and_fetch<R: Repository>(repo: &mut R, item: R::Item) -> Result<(), R::Error>
where
    R::Item: std::fmt::Debug,
    R::Id: std::fmt::Debug,
{
    let id = repo.insert(item)?;
    println!("Inserted with id: {id:?}");
    let retrieved = repo.get(&id)?;
    println!("Retrieved: {retrieved:?}");
    Ok(())
}

fn main() {
    let mut repo = InMemoryUserRepo::new();
    create_and_fetch(&mut repo, User {
        name: "Alice".into(),
        email: "alice@example.com".into(),
    }).unwrap();
}
```

</details>

---

### Упражнение 7: безопасная обертка над Unsafe (Ch11) ★★★ (~45 мин)

Напишите структуру `FixedVec<T, const N: usize>`, представляющую собой стек-аллоцированный вектор фиксированной емкости.
Требования:
- Метод `push(&mut self, value: T) -> Result<(), T>` возвращает `Err(value)`, если вектор заполнен
- Метод `pop(&mut self) -> Option<T>` возвращает и удаляет последний элемент
- Метод `as_slice(&self) -> &[T]` заимствует инициализированные элементы
- Все публичные методы должны быть безопасными; любой блок `unsafe` должен быть задокументирован комментариями `SAFETY:`
- Реализация `Drop` должна корректно очищать инициализированные элементы

**Подсказка**: Используйте `MaybeUninit<T>` и `[const { MaybeUninit::uninit() }; N]`.

<details>
<summary>🔑 Решение</summary>

```rust
use std::mem::MaybeUninit;

pub struct FixedVec<T, const N: usize> {
    data: [MaybeUninit<T>; N],
    len: usize,
}

impl<T, const N: usize> FixedVec<T, N> {
    pub fn new() -> Self {
        FixedVec {
            data: [const { MaybeUninit::uninit() }; N],
            len: 0,
        }
    }

    pub fn push(&mut self, value: T) -> Result<(), T> {
        if self.len >= N { return Err(value); }
        // SAFETY: len < N, поэтому data[len] находится в границах массива.
        self.data[self.len] = MaybeUninit::new(value);
        self.len += 1;
        Ok(())
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 { return None; }
        self.len -= 1;
        // SAFETY: элемент data[len] был инициализирован (len был > 0 до декремента).
        Some(unsafe { self.data[self.len].assume_init_read() })
    }

    pub fn as_slice(&self) -> &[T] {
        // SAFETY: все элементы в диапазоне data[0..len] инициализированы, а MaybeUninit<T>
        // имеет такое же представление в памяти, как и T.
        unsafe { std::slice::from_raw_parts(self.data.as_ptr() as *const T, self.len) }
    }

    pub fn len(&self) -> usize { self.len }
    pub fn is_empty(&self) -> bool { self.len == 0 }
}

impl<T, const N: usize> Drop for FixedVec<T, N> {
    fn drop(&mut self) {
        // SAFETY: элементы data[0..len] инициализированы — сбрасываем каждый из них.
        for i in 0..self.len {
            unsafe { self.data[i].assume_init_drop(); }
        }
    }
}

fn main() {
    let mut v = FixedVec::<String, 4>::new();
    v.push("hello".into()).unwrap();
    v.push("world".into()).unwrap();
    assert_eq!(v.as_slice(), &["hello", "world"]);
    assert_eq!(v.pop(), Some("world".into()));
    assert_eq!(v.len(), 1);
    // Drop очищает оставшуюся строку "hello"
}
```

</details>

---

### Упражнение 8: декларативный макрос — `map!` (Ch12) ★ (~15 мин)

Напишите макрос `map!`, создающий карту `HashMap` из пар «ключ-значение» аналогично макросу `vec![]`:

```rust
let m = map! {
    "host" => "localhost",
    "port" => "8080",
};
assert_eq!(m.get("host"), Some(&"localhost"));
assert_eq!(m.len(), 2);
```

Требования:
- Поддержка висящей запятой (trailing comma)
- Поддержка пустого вызова `map!{}`
- Совместимость с любыми типами, реализующими `Into<K>` and `Into<V>` для максимальной гибкости

<details>
<summary>🔑 Решение</summary>

```rust
macro_rules! map {
    // Случай пустого макроса
    () => {
        std::collections::HashMap::new()
    };
    // Одна или более пар ключ => значение (висящая запятая опциональна)
    ( $( $key:expr => $val:expr ),+ $(,)? ) => {{
        let mut m = std::collections::HashMap::new();
        $( m.insert($key, $val); )+
        m
    }};
}

fn main() {
    // Базовое использование:
    let config = map! {
        "host" => "localhost",
        "port" => "8080",
        "timeout" => "30",
    };
    assert_eq!(config.len(), 3);
    assert_eq!(config["host"], "localhost");

    // Пустая карта:
    let empty: std::collections::HashMap<String, String> = map!();
    assert!(empty.is_empty());

    // Разные типы:
    let scores = map! {
        1 => 100,
        2 => 200,
    };
    assert_eq!(scores[&1], 100);
}
```

</details>

---

### Упражнение 9: кастомная десериализация в serde (Ch10) ★★★ (~45 мин)

Разработайте структуру-обертку `Duration`, которая десериализуется из удобочитаемых строк вида `"30s"`, `"5m"`, `"2h"` с помощью кастомного десериализатора `serde`. Данная структура также должна поддерживать обратную сериализацию в тот же строковый формат.

<details>
<summary>🔑 Решение</summary>

```rust,ignore
use serde::{Deserialize, Deserializer, Serialize, Serializer};
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
struct HumanDuration(std::time::Duration);

impl HumanDuration {
    fn from_str(s: &str) -> Result<Self, String> {
        let s = s.trim();
        if s.is_empty() { return Err("пустая строка длительности".into()); }

        let (num_str, suffix) = s.split_at(
            s.find(|c: char| !c.is_ascii_digit()).unwrap_or(s.len())
        );
        let value: u64 = num_str.parse()
            .map_err(|_| format!("неверное число: {num_str}"))?;

        let duration = match suffix {
            "s" | "sec"  => std::time::Duration::from_secs(value),
            "m" | "min"  => std::time::Duration::from_secs(value * 60),
            "h" | "hr"   => std::time::Duration::from_secs(value * 3600),
            "ms"         => std::time::Duration::from_millis(value),
            other        => return Err(format!("неизвестный суффикс: {other}")),
        };
        Ok(HumanDuration(duration))
    }
}

impl fmt::Display for HumanDuration {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let secs = self.0.as_secs();
        if secs == 0 {
            write!(f, "{}ms", self.0.as_millis())
        } else if secs % 3600 == 0 {
            write!(f, "{}h", secs / 3600)
        } else if secs % 60 == 0 {
            write!(f, "{}m", secs / 60)
        } else {
            write!(f, "{}s", secs)
        }
    }
}

impl Serialize for HumanDuration {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        serializer.serialize_str(&self.to_string())
    }
}

impl<'de> Deserialize<'de> for HumanDuration {
    fn deserialize<D: Deserializer<'de>>(deserializer: D) -> Result<Self, D::Error> {
        let s = String::deserialize(deserializer)?;
        HumanDuration::from_str(&s).map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    timeout: HumanDuration,
    retry_interval: HumanDuration,
}

fn main() {
    let json = r#"{ "timeout": "30s", "retry_interval": "5m" }"#;
    let config: Config = serde_json::from_str(json).unwrap();

    assert_eq!(config.timeout.0, std::time::Duration::from_secs(30));
    assert_eq!(config.retry_interval.0, std::time::Duration::from_secs(300));

    // Корректно сериализуется обратно (round-trip):
    let serialized = serde_json::to_string(&config).unwrap();
    assert!(serialized.contains("30s"));
    assert!(serialized.contains("5m"));
    println!("Config: {serialized}");
}
```

</details>

### Упражнение 10 — конкурентный загрузчик с таймаутом ★★ (~25 мин)

Напишите асинхронную функцию `fetch_all`, запускающую три задачи через `tokio::spawn`, каждая из которых имитирует сетевой запрос с помощью `tokio::time::sleep`. Объедините выполнение всех трех задач с помощью `tokio::try_join!`, обернутого в `tokio::time::timeout(Duration::from_secs(5), ...)`. Метод должен возвращать `Result<Vec<String>, ...>` или ошибку, если какая-либо задача завершилась сбоем либо истек срок ожидания.

**Цели обучения**: изучение `tokio::spawn`, `try_join!`, `timeout`, распространение ошибок через границы задач.

<details>
<summary>Подсказка</summary>

Каждая запущенная задача возвращает `Result<String, _>`. Макрос `try_join!` распаковывает все три результата. Оберните весь `try_join!` в вызов `timeout()` — ошибка `Elapsed` будет означать достижение крайнего срока времени ожидания.

</details>

<details>
<summary>Решение</summary>

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
    .await??; // первый ? = timeout, второй ? = join

    Ok(vec![a?, b?, c?]) // распаковываем внутренние Results
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

### Упражнение 11 — асинхронный конвейер на каналах ★★★ (~40 мин)

Постройте конвейер производитель → преобразователь → потребитель с использованием `tokio::sync::mpsc`:

1. **Производитель (Producer)**: отправляет целые числа от 1 до 20 в канал А (емкость 4).
2. **Преобразователь (Transformer)**: читает данные из канала А, возводит каждое значение в квадрат и отправляет результат в канал Б.
3. **Потребитель (Consumer)**: читает данные из канала Б, собирает их в вектор `Vec<u64>` и возвращает его.

Все три этапа должны выполняться как конкурентные задачи через `tokio::spawn`. Используйте ограниченные (bounded) каналы для демонстрации механизма противодавления (back-pressure). Убедитесь с помощью `assert_eq!`, что итоговый вектор равен `[1, 4, 9, ..., 400]`.

**Цели обучения**: работа с `mpsc::channel`, противодавление в ограниченных каналах, `tokio::spawn` с замыканиями `move`, корректное завершение работы через закрытие канала.

<details>
<summary>Решение</summary>

```rust,ignore
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx_a, mut rx_a) = mpsc::channel::<u64>(4); // ограниченный канал — противодавление
    let (tx_b, mut rx_b) = mpsc::channel::<u64>(4);

    // Производитель
    let producer = tokio::spawn(async move {
        for i in 1..=20u64 {
            tx_a.send(i).await.unwrap();
        }
        // tx_a сбрасывается здесь → канал A закрывается
    });

    // Преобразователь
    let transformer = tokio::spawn(async move {
        while let Some(val) = rx_a.recv().await {
            tx_b.send(val * val).await.unwrap();
        }
        // tx_b сбрасывается здесь → канал B закрывается
    });

    // Потребитель
    let consumer = tokio::spawn(async move {
        let mut results = Vec::new();
        while let Some(val) = rx_b.recv().await {
            results.push(val);
        }
        results
    });

    producer.await.unwrap();
    transformer.await.unwrap();
    let results = consumer.await.unwrap();

    let expected: Vec<u64> = (1..=20).map(|x: u64| x * x).collect();
    assert_eq!(results, expected);
    println!("Pipeline complete: {results:?}");
}
```

</details>

***
