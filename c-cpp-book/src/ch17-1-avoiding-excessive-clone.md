## Как избегать избыточного использования clone()

> **Чему вы научитесь:** Почему `.clone()` считается «запахом кода» (code smell) в Rust, как реструктурировать владение для устранения ненужного копирования и какие конкретные шаблоны сигнализируют о проблемах проектирования владения.

- Для пришедших из C++ метод `.clone()` кажется безопасным выбором по умолчанию — «просто скопируй это». Однако избыточное клонирование маскирует проблемы с владением и снижает производительность.
- **Эмпирическое правило**: если вы используете клонирование только для того, чтобы задобрить проверку заимствований (borrow checker), вам, скорее всего, нужно перепроектировать схему владения.

### Когда clone() используется некорректно

```rust
// ПЛОХО: клонирование String только для передачи в функцию, которая его только читает
fn log_message(msg: String) {  // Ненужное взятие во владение
    println!("[LOG] {}", msg);
}
let message = String::from("GPU test passed");
log_message(message.clone());  // Расточительно: выделяет память под совершенно новую строку String
log_message(message);           // Оригинал поглощен — клонирование было бессмысленным
```

```rust
// ХОРОШО: принимаем заимствование — ноль аллокаций
fn log_message(msg: &str) {    // Заимствует, не владеет
    println!("[LOG] {}", msg);
}
let message = String::from("GPU test passed");
log_message(&message);          // Без клонирования, без аллокаций
log_message(&message);          // Можно вызвать снова — переменная message не поглощена
```

### Реальный пример: возврат `&str` вместо клонирования
```rust
// Пример: healthcheck.rs — возвращает заимствованное представление, ноль аллокаций
pub fn serial_or_unknown(&self) -> &str {
    self.serial.as_deref().unwrap_or(UNKNOWN_VALUE)
}

pub fn model_or_unknown(&self) -> &str {
    self.model.as_deref().unwrap_or(UNKNOWN_VALUE)
}
```
Эквивалент в C++ возвращал бы `const std::string&` или `std::string_view`, но в C++ ни один из этих вариантов не проверяется на время жизни. В Rust компилятор гарантирует, что возвращенный срез `&str` не может пережить `self`.

### Реальный пример: статические срезы строк — без использования кучи
```rust
// Пример: healthcheck.rs — таблицы строк на этапе компиляции
const HBM_SCREEN_RECIPES: &[&str] = &[
    "hbm_ds_ntd", "hbm_ds_ntd_gfx", "hbm_dt_ntd", "hbm_dt_ntd_gfx",
    "hbm_burnin_8h", "hbm_burnin_24h",
];
```
В C++ это обычно представляло бы собой `std::vector<std::string>` (с выделением памяти в куче при первом использовании). В Rust `&'static [&'static str]` живет в памяти, доступной только для чтения — нулевые затраты во время выполнения.

### Когда clone() ДЕЙСТВИТЕЛЬНО уместна

| **Ситуация** | **Почему клонирование допустимо** | **Пример** |
|--------------|--------------------|-----------|
| `Arc::clone()` для многопоточности | Увеличивает счетчик ссылок (~1 нс), не копирует данные | `let flag = stop_flag.clone();` |
| Передача данных в порожденный поток | Потоку нужна своя копия | `let ctx = ctx.clone(); thread::spawn(move \|\| { ... })` |
| Извлечение полей из `&self` | Нельзя переместить данные из заимствованного объекта | `self.name.clone()` при возврате владеемой `String` |
| Мелкие типы с `Copy`, обернутые в `Option` | `.copied()` нагляднее, чем `.clone()` | `opt.get(0).copied()` для `Option<&u32>` → `Option<u32>` |

### Реальный пример: Arc::clone для разделения ресурсов между потоками
```rust
// Пример: workload.rs — Arc::clone дешев (увеличивает счетчик ссылок)
let stop_flag = Arc::new(AtomicBool::new(false));
let stop_flag_clone = stop_flag.clone();   // ~1 нс, данные не копируются
let ctx_clone = ctx.clone();               // Клонируем контекст для перемещения в поток

let sensor_handle = thread::spawn(move || {
    // ...использует stop_flag_clone и ctx_clone
});
```

### Чек-лист: стоит ли клонировать?
1. **Могу ли я принять `&str` / `&T` вместо `String` / `T`?** → Заимствуйте, а не клонируйте.
2. **Могу ли я перестроить структуру так, чтобы избежать потребности в двух владельцах?** → Передавайте по ссылке или используйте блоки видимости (scopes).
3. **Является ли это `Arc::clone()`?** → Это нормально, сложность O(1).
4. **Перемещаю ли я данные в поток или замыкание?** → Клонирование необходимо.
5. **Вызывается ли клонирование в «горячем» цикле?** → Выполните профилирование и рассмотрите заимствование или `Cow<T>`.

----

## `Cow<'a, T>`: Clone-on-Write — заимствуйте, когда можете, клонируйте, когда необходимо

`Cow` (Clone on Write, клонирование при записи) — это перечисление (enum), которое хранит **либо** заимствованную ссылку, **либо** владеемое значение. Это эквивалент Rust для подхода «избегайте выделения памяти, когда это возможно, но выделяйте её, если требуется модификация». В C++ нет прямого аналога; ближайшим эквивалентом была бы функция, которая в некоторых сценариях возвращает `const std::string&`, а в других — `std::string`.

### Зачем нужен `Cow`

```rust
// Без Cow вам приходится выбирать: всегда заимствовать ИЛИ всегда клонировать
fn normalize(s: &str) -> String {          // Всегда выделяет память!
    if s.contains(' ') {
        s.replace(' ', "_")               // Новая строка String (требуется выделение памяти)
    } else {
        s.to_string()                     // Ненужное выделение памяти!
    }
}

// С Cow — заимствуем, когда строка не изменена, выделяем память только при модификации
use std::borrow::Cow;

fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(' ') {
        Cow::Owned(s.replace(' ', "_"))    // Выделяет память (требуется изменить)
    } else {
        Cow::Borrowed(s)                   // Ноль аллокаций (проброс ссылки)
    }
}
```

### Как работает `Cow`

```rust
use std::borrow::Cow;

// Cow<'a, str> по сути представляет собой:
// enum Cow<'a, str> {
//     Borrowed(&'a str),     // Ссылка с нулевой стоимостью
//     Owned(String),          // Владеемое значение, выделенное в куче
// }

fn greet(name: &str) -> Cow<'_, str> {
    if name.is_empty() {
        Cow::Borrowed("stranger")         // Статическая строка — без выделения памяти
    } else if name.starts_with(' ') {
        Cow::Owned(name.trim().to_string()) // Изменено — требуется выделение памяти
    } else {
        Cow::Borrowed(name)               // Проброс ссылки — без выделения памяти
    }
}

fn main() {
    let g1 = greet("Alice");     // Cow::Borrowed("Alice")
    let g2 = greet("");          // Cow::Borrowed("stranger")
    let g3 = greet(" Bob ");     // Cow::Owned("Bob")
    
    // Cow<str> реализует Deref<Target = str>, поэтому его можно использовать как &str:
    println!("Hello, {g1}!");    // Работает — Cow автоматически разыменовывается в &str
    println!("Hello, {g2}!");
    println!("Hello, {g3}!");
}
```

### Реальный сценарий использования: нормализация значений конфигурации

```rust
use std::borrow::Cow;

/// Нормализация имени SKU: обрезка пробелов, приведение к нижнему регистру.
/// Возвращает Cow::Borrowed, если имя уже нормализовано (ноль аллокаций).
fn normalize_sku(sku: &str) -> Cow<'_, str> {
    let trimmed = sku.trim();
    if trimmed == sku && sku.chars().all(|c| c.is_lowercase() || !c.is_alphabetic()) {
        Cow::Borrowed(sku)   // Уже нормализовано — без выделения памяти
    } else {
        Cow::Owned(trimmed.to_lowercase())  // Требует изменения — выделяем память
    }
}

fn main() {
    let s1 = normalize_sku("server-x1");   // Borrowed — ноль аллокаций
    let s2 = normalize_sku("  Server-X1 "); // Owned — необходимо выделить память
    println!("{s1}, {s2}"); // "server-x1, server-x1"
}
```

### Когда использовать `Cow`

| **Ситуация** | **Использовать `Cow`?** |
|--------------|---------------|
| Функция большую часть времени возвращает ввод без изменений | ✅ Да — избегаем ненужного клонирования |
| Парсинг/нормализация строк (trim, lowercase, replace) | ✅ Да — часто ввод уже валиден |
| Постоянная модификация — любая ветка кода выделяет память | ❌ Нет — просто возвращайте `String` |
| Простой проброс (никогда не изменяется) | ❌ Нет — просто возвращайте `&str` |
| Данные хранятся в структуре долгосрочно | ❌ Нет — используйте `String` (владение) |

> **Сравнение с C++**: `Cow<str>` похож на функцию, возвращающую `std::variant<std::string_view, std::string>`, но с автоматическим разыменованием и без необходимости писать шаблонный код для доступа к значению.

----

## `Weak<T>`: Разрыв циклов ссылок — аналог `weak_ptr` в Rust

`Weak<T>` — это эквивалент `std::weak_ptr<T>` из C++ в Rust. Он хранит невладеющую ссылку на значение `Rc<T>` или `Arc<T>`. Значение может быть удалено из памяти, пока ссылки `Weak` все еще существуют — вызов `upgrade()` вернет `None`, если значение уже освобождено.

### Зачем нужен `Weak`

`Rc<T>` и `Arc<T>` создают циклы ссылок, если два значения указывают друг на друга. Ни одно из них никогда не достигнет счетчика ссылок 0, поэтому ни одно не будет удалено (утечка памяти). `Weak` разрывает этот цикл:

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: String,
    parent: RefCell<Weak<Node>>,      // Weak — не мешает удалению родительского узла
    children: RefCell<Vec<Rc<Node>>>,  // Strong — родитель владеет дочерними узлами
}

impl Node {
    fn new(value: &str) -> Rc<Node> {
        Rc::new(Node {
            value: value.to_string(),
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(Vec::new()),
        })
    }

    fn add_child(parent: &Rc<Node>, child: &Rc<Node>) {
        // Потомок получает слабую ссылку (weak reference) на родителя (без цикла)
        *child.parent.borrow_mut() = Rc::downgrade(parent);
        // Родитель получает сильную ссылку (strong reference) на потомка
        parent.children.borrow_mut().push(Rc::clone(child));
    }
}

fn main() {
    let root = Node::new("root");
    let child = Node::new("child");
    Node::add_child(&root, &child);

    // Получаем доступ к родителю из потомка через upgrade()
    if let Some(parent) = child.parent.borrow().upgrade() {
        println!("Child's parent: {}", parent.value); // "root"
    }
    
    println!("Root strong count: {}", Rc::strong_count(&root));  // 1
    println!("Root weak count: {}", Rc::weak_count(&root));      // 1
}
```

### Сравнение с C++

```cpp
// C++ — weak_ptr для разрыва цикла shared_ptr
struct Node {
    std::string value;
    std::weak_ptr<Node> parent;                  // Weak — без владения
    std::vector<std::shared_ptr<Node>> children;  // Strong — владеет потомками

    static auto create(const std::string& v) {
        return std::make_shared<Node>(Node{v, {}, {}});
    }
};

auto root = Node::create("root");
auto child = Node::create("child");
child->parent = root;          // присваивание weak_ptr
root->children.push_back(child);

if (auto p = child->parent.lock()) {   // lock() → shared_ptr или null
    std::cout << "Parent: " << p->value << std::endl;
}
```

| C++ | Rust | Примечания |
|-----|------|-------|
| `shared_ptr<T>` | `Rc<T>` (однопоточный) / `Arc<T>` (многопоточный) | Та же семантика |
| `weak_ptr<T>` | `Weak<T>` из `Rc::downgrade()` / `Arc::downgrade()` | Та же семантика |
| `weak_ptr::lock()` → `shared_ptr` или null | `Weak::upgrade()` → `Option<Rc<T>>` | `None`, если объект удален |
| `shared_ptr::use_count()` | `Rc::strong_count()` | Тот же смысл |

### Когда использовать `Weak`

| **Ситуация** | **Шаблон** |
|--------------|-----------|
| Связи «родитель ↔ потомок» в дереве | Родитель хранит `Rc<Child>`, потомок хранит `Weak<Parent>` |
| Паттерн «Наблюдатель» (Observer) / слушатели событий | Источник событий хранит `Weak<Observer>`, наблюдатель хранит `Rc<Source>` |
| Кэш, не препятствующий освобождению памяти | `HashMap<Key, Weak<Value>>` — записи устаревают естественным путем |
| Разрыв циклов в графовых структурах | Перекрестные связи используют `Weak`, ребра дерева — `Rc`/`Arc` |

> **Отдавайте предпочтение паттерну «Арена»** (практический пример 2) вместо `Rc/Weak` для древовидных структур в новом коде. `Vec<T>` + индексы проще, быстрее и не несут накладных расходов на подсчет ссылок. Используйте `Rc/Weak`, когда требуется совместное владение с динамическим временем жизни.

----

## Copy против Clone, PartialEq против Eq — когда и что выводить (derive)

- **Copy ≈ тривиально копируемый тип в C++ (без пользовательского конструктора копирования/деструктора).** Такие типы, как `int`, `enum` и простые структуры POD (простые данные), компилятор копирует побитово (`memcpy`) автоматически. В Rust концепция `Copy` аналогична: присваивание `let b = a;` выполняет неявное побитовое копирование, после чего обе переменные остаются валидными.
- **Clone ≈ конструктор копирования в C++ / глубокое копирование через `operator=`.** Если класс C++ содержит пользовательский конструктор копирования (например, для глубокого копирования члена `std::vector`), то в Rust эквивалентом является реализация `Clone`. Вам необходимо вызывать метод `.clone()` явно — Rust никогда не скрывает дорогостоящее копирование за оператором `=`.
- **Ключевое отличие:** в C++ как тривиальное, так и глубокое копирование выполняются неявно с помощью одного и того же синтаксиса `=`. Rust заставляет вас делать выбор: типы с типажом `Copy` копируются незаметно (дешево), типы без типажа `Copy` по умолчанию **перемещаются** (move), а для выполнения дорогостоящего дублирования данных вы должны явно вызвать `.clone()`.
- Аналогично, оператор `operator==` в C++ не делает различий между типами, для которых равенство `a == a` выполняется всегда (как целые числа), и типами, для которых это не так (как `float` с его NaN). В Rust это различие формализовано разделением на типажи `PartialEq` и `Eq`.

### Copy против Clone

| | **Copy** | **Clone** |
|---|---------|----------|
| **Как работает** | Побитовое копирование memcpy (неявно) | Пользовательская логика (явный вызов `.clone()`) |
| **Когда происходит** | При присваивании: `let b = a;` | Только при вызове `.clone()` |
| **После копирования/клонирования** | Обе переменные `a` и `b` валидны | Обе переменные `a` и `b` валидны |
| **Без использования обоих** | `let b = a;` **перемещает** `a` (доступ к `a` теряется) | `let b = a;` **перемещает** `a` (доступ к `a` теряется) |
| **Разрешено для** | Типов без данных в куче | Любых типов |
| **Аналогия в C++** | Тривиально копируемые / POD-типы | Конструктор копирования (глубокое копирование) |

### Реальный пример: Copy — простые перечисления

```rust
// Из fan_diag/src/sensor.rs — все варианты не содержат данных, умещается в 1 байт
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Default)]
pub enum FanStatus {
    #[default]
    Normal,
    Low,
    High,
    Missing,
    Failed,
    Unknown,
}

let status = FanStatus::Normal;
let copy = status;   // Неявное копирование — переменная status все еще валидна
println!("{:?} {:?}", status, copy);  // Обе переменные работают
```

### Реальный пример: Copy — перечисление с целочисленной полезной нагрузкой

```rust
// Пример: healthcheck.rs — варианты с u32 поддерживают Copy, поэтому и все перечисление тоже
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum HealthcheckStatus {
    Pass,
    ProgramError(u32),
    DmesgError(u32),
    RasError(u32),
    OtherError(u32),
    Unknown,
}
```

### Реальный пример: только Clone — структура с данными в куче

```rust
// Пример: components.rs — наличие String препятствует выводу Copy
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FruData {
    pub technology: DeviceTechnology,
    pub physical_location: String,      // ← String: выделяется в куче, скопировать через Copy нельзя
    pub expected: bool,
    pub removable: bool,
}
// let a = fru_data;   → ПЕРЕМЕЩАЕТ (доступ к fru_data теряется)
// let a = fru_data.clone();  → КЛОНИРУЕТ (fru_data все еще валидна, новая аллокация в куче)
```

### Правило: может ли тип поддерживать Copy?

```text
Содержит ли тип String, Vec, Box, HashMap,
Rc, Arc или любой другой тип, владеющий памятью в куче?
    ДА → Только Clone (не может реализовывать Copy)
    НЕТ → Вы МОЖЕТЕ автоматически вывести Copy (и вам следует это сделать, если тип небольшой)
```

### PartialEq против Eq

| | **PartialEq** | **Eq** |
|---|--------------|-------|
| **Что это дает** | Операторы `==` и `!=` | Маркер: «отношение равенства рефлексивно» |
| **Рефлексивность? (a == a)** | Не гарантируется | **Гарантируется** |
| **Почему это важно** | `f32::NAN != f32::NAN` | Для ключей `HashMap` **требуется** `Eq` |
| **Когда выводить (derive)** | Почти всегда | Когда у типа нет полей типа `f32`/`f64` |
| **Аналогия в C++** | `operator==` | Нет прямого аналога (C++ не проверяет рефлексивность) |

### Реальный пример: Eq — использование в качестве ключа HashMap

```rust
// Из hms_trap/src/cpu_handler.rs — Hash требует наличия Eq
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum CpuFaultType {
    InvalidFaultType,
    CpuCperFatalErr,
    CpuLpddr5UceErr,
    CpuC2CUceFatalErr,
    // ...
}
// Используется как: HashMap<CpuFaultType, FaultHandler>
// Ключи HashMap должны реализовывать Eq + Hash — с одним лишь PartialEq код не скомпилируется
```

### Реальный пример: Eq невозможен — тип содержит f32

```rust
// Пример: types.rs — f32 препятствует выводу Eq
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct TemperatureSensors {
    pub warning_threshold: Option<f32>,   // ← f32 содержит NaN, для которого NaN ≠ NaN
    pub critical_threshold: Option<f32>,  // ← невозможно автоматически вывести Eq
    pub sensor_names: Vec<String>,
}
// Не может использоваться в качестве ключа HashMap. Нельзя автоматически вывести Eq.
// Причина: f32::NAN == f32::NAN ложно, что нарушает рефлексивность.
```

### PartialOrd против Ord

| | **PartialOrd** | **Ord** |
|---|---------------|--------|
| **Что это дает** | Операторы `<`, `>`, `<=`, `>=` | Метод `.sort()`, ключи для `BTreeMap` |
| **Полный порядок?** | Нет (некоторые пары могут быть несравнимы) | **Да** (любые две величины сравнимы) |
| **f32/f64?** | Только `PartialOrd` (NaN нарушает порядок) | Автоматически вывести `Ord` нельзя |

### Реальный пример: Ord — ранжирование важности ошибок

```rust
// Из hms_trap/src/fault.rs — порядок вариантов определяет важность
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum FaultSeverity {
    Info,      // наименьшая (дискриминант 0)
    Warning,   //            (дискриминант 1)
    Error,     //            (дискриминант 2)
    Critical,  // наибольшая  (дискриминант 3)
}
// FaultSeverity::Info < FaultSeverity::Critical → true
// Позволяет делать так: if severity >= FaultSeverity::Error { escalate(); }
```

### Реальный пример: Ord — диагностические уровни для сравнения

```rust
// Пример: orchestration.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Default)]
pub enum GpuDiagLevel {
    #[default]
    Quick,     // наименьший
    Standard,
    Extended,
    Full,      // наибольший
}
// Позволяет делать так: if requested_level >= GpuDiagLevel::Extended { run_extended_tests(); }
```

### Дерево решений для derive

```text
                         Ваш новый тип
                               │
                    Содержит String/Vec/Box?
                       /              \
                     ДА               НЕТ
                      │                 │
                Только Clone      Clone + Copy
                      │                 │
               Содержит f32/f64?  Содержит f32/f64?
                 /          \        /          \
               ДА           НЕТ    ДА           НЕТ
                │            │      │            │
          Только PartialEq PartialEq Только PartialEq PartialEq
                            + Eq                   + Eq
                             │                      │
                       Нужна сортировка?      Нужна сортировка?
                         /       \              /       \
                       ДА        НЕТ          ДА        НЕТ
                        │         │            │         │
                  PartialOrd    Готово   PartialOrd    Готово
                  + Ord                  + Ord
                        │                      │
                  Нужен как              Нужен как
                  ключ map?              ключ map?
                     │                      │
                   + Hash                 + Hash
```

### Быстрый справочник: частые комбинации derive из реального кода на Rust

| **Категория типа** | **Типичный derive** | **Пример** |
|-------------------|--------------------|------------|
| Простые перечисления статусов | `Copy, Clone, PartialEq, Eq, Default` | `FanStatus` |
| Перечисления для ключей HashMap | `Copy, Clone, PartialEq, Eq, Hash` | `CpuFaultType`, `SelComponent` |
| Сравнимые перечисления важности | `Copy, Clone, PartialEq, Eq, PartialOrd, Ord` | `FaultSeverity`, `GpuDiagLevel` |
| Структуры данных со строками | `Clone, Debug, Serialize, Deserialize` | `FruData`, `OverallSummary` |
| Сериализуемая конфигурация | `Clone, Debug, Default, Serialize, Deserialize` | `DiagConfig` |

----
