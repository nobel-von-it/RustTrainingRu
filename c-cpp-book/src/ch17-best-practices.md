# Краткое руководство по лучшим практикам Rust

> **Чему вы научитесь:** Практическим рекомендациям по написанию идиоматичного кода на Rust — организации кода, соглашениям об именовании, шаблонам обработки ошибок и документированию. Глава-справочник, к которой вы будете часто возвращаться.

## Организация кода
- **Отдавайте предпочтение коротким функциям**: их легче тестировать и анализировать.
- **Используйте понятные и точные имена**: `calculate_total_price()` вместо `calc()`.
- **Группируйте связанную функциональность**: используйте модули и отдельные файлы.
- **Пишите документацию**: используйте `///` для публичных API.

## Обработка ошибок
- **Избегайте `unwrap()`, кроме случаев, когда ошибка невозможна**: используйте его только тогда, когда вы на 100% уверены, что паники не произойдет.
```rust
// Плохо: может вызвать панику
let value = some_option.unwrap();

// Хорошо: обрабатываем вариант None
let value = some_option.unwrap_or(default_value);
let value = some_option.unwrap_or_else(|| expensive_computation());
let value = some_option.unwrap_or_default(); // Использует типаж Default

// Для Result<T, E>
let value = some_result.unwrap_or(fallback_value);
let value = some_result.unwrap_or_else(|err| {
    eprintln!("Произошла ошибка: {err}");
    default_value
});
```
- **Используйте `expect()` с информативными сообщениями**: когда применение unwrap оправдано, объясните почему.
```rust
let config = std::env::var("CONFIG_PATH")
    .expect("переменная окружения CONFIG_PATH должна быть установлена");
```
- **Возвращайте `Result<T, E>` для операций, которые могут завершиться ошибкой**: пусть вызывающий код решает, как обрабатывать ошибки.
- **Используйте `thiserror` для пользовательских типов ошибок**: это более эргономично, чем ручная реализация.
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("Ошибка ввода-вывода: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Ошибка парсинга: {message}")]
    Parse { message: String },
    
    #[error("Значение {value} находится вне диапазона")]
    OutOfRange { value: i32 },
}
```
- **Объединяйте ошибки в цепочки с помощью оператора `?`**: распространяйте ошибки вверх по стеку вызовов.
- **Отдавайте предпочтение `thiserror` перед `anyhow`**: в нашей команде принято определять явные перечисления ошибок с помощью `#[derive(thiserror::Error)]`, чтобы вызывающая сторона могла сопоставлять конкретные варианты ошибок. `anyhow::Error` удобен для быстрого прототипирования, но он стирает исходный тип ошибки, усложняя для вызывающего кода обработку конкретных сбоев. Используйте `thiserror` для библиотечного и промышленного (production) кода; оставьте `anyhow` для временных скриптов или бинарных файлов верхнего уровня, где требуется только вывести ошибку.
- **Когда `unwrap()` допустим**:
  - **Модульные тесты (Unit tests)**: `assert_eq!(result.unwrap(), expected)`
  - **Прототипирование**: черновой код, который вы впоследствии перепишете
  - **Гарантированно успешные операции**: когда вы можете доказать, что операция не завершится ошибкой
```rust
let numbers = vec![1, 2, 3];
let first = numbers.get(0).unwrap(); // Безопасно: мы только что создали вектор с элементами

// Лучше: используйте expect() с пояснением
let first = numbers.get(0).expect("вектор numbers не пуст по построению");
```
- **Быстрый отказ (Fail fast)**: проверяйте предусловия на ранней стадии и немедленно возвращайте ошибки.

## Управление памятью
- **Предпочитайте заимствование клонированию**: по возможности используйте `&T` вместо клонирования.
- **Используйте `Rc<T>` с осторожностью**: только когда вам действительно нужно совместное владение.
- **Ограничивайте время жизни**: используйте блоки областей видимости `{}` для контроля момента удаления (drop) значений.
- **Избегайте `RefCell<T>` в публичных API**: сохраняйте внутреннюю мутабельность (interior mutability) скрытой внутри модуля.

## Производительность
- **Профилируйте перед оптимизацией**: используйте `cargo bench` и инструменты профилирования.
- **Предпочитайте итераторы циклам**: они более читаемы и зачастую быстрее.
- **Используйте `&str` вместо `String`**: когда вам не нужно владение строкой.
- **Рассмотрите `Box<T>` для больших объектов на стеке**: при необходимости переносите их в кучу.

## Важнейшие типажи для реализации

### Базовые типажи, которые стоит реализовать для каждого типа

При создании пользовательских типов рассмотрите возможность реализации этих фундаментаных типажей, чтобы ваши типы воспринимались как нативные для Rust:

#### **Debug и Display**
```rust
use std::fmt;

#[derive(Debug)]  // Автоматическая реализация для отладки
struct Person {
    name: String,
    age: u32,
}

// Ручная реализация Display для пользовательского вывода
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

// Использование:
let person = Person { name: "Alice".to_string(), age: 30 };
println!("{:?}", person);  // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);    // Display: Alice (age 30)
```

#### **Clone и Copy**
```rust
// Copy: неявное дублирование для небольших простых типов
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

// Clone: явное дублирование для сложных типов
#[derive(Debug, Clone)]
struct Person {
    name: String,  // String не реализует Copy
    age: u32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Copy (неявно)

let person1 = Person { name: "Bob".to_string(), age: 25 };
let person2 = person1.clone();  // Clone (явно)
```

#### **PartialEq и Eq**
```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, PartialEq)]
struct Temperature {
    celsius: f64,  // f64 не реализует Eq (из-за NaN)
}

let id1 = UserId(123);
let id2 = UserId(123);
assert_eq!(id1, id2);  // Работает благодаря PartialEq

let temp1 = Temperature { celsius: 20.0 };
let temp2 = Temperature { celsius: 20.0 };
assert_eq!(temp1, temp2);  // Работает с PartialEq
```

#### **PartialOrd и Ord**
```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

let high = Priority(1);
let low = Priority(10);
assert!(high < low);  // Меньшие числа = более высокий приоритет

// Использование в коллекциях
let mut priorities = vec![Priority(5), Priority(1), Priority(8)];
priorities.sort();  // Работает, так как Priority реализует Ord
```

#### **Default**
```rust
#[derive(Debug, Default)]
struct Config {
    debug: bool,           // false (по умолчанию)
    max_connections: u32,  // 0 (по умолчанию)
    timeout: Option<u64>,  // None (по умолчанию)
}

// Пользовательская реализация Default
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            max_connections: 100,  // Пользовательское значение по умолчанию
            timeout: Some(30),     // Пользовательское значение по умолчанию
        }
    }
}

let config = Config::default();
let config = Config { debug: true, ..Default::default() };  // Частичное переопределение
```

#### **From и Into**
```rust
struct UserId(u64);
struct UserName(String);

// Реализуйте From, а Into вы получите автоматически
impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

impl From<String> for UserName {
    fn from(name: String) -> Self {
        UserName(name)
    }
}

impl From<&str> for UserName {
    fn from(name: &str) -> Self {
        UserName(name.to_string())
    }
}

// Использование:
let user_id: UserId = 123u64.into();         // Использование Into
let user_id = UserId::from(123u64);          // Использование From
let username = UserName::from("alice");      // &str -> UserName
let username: UserName = "bob".into();       // Использование Into
```

#### **TryFrom и TryInto**
```rust
use std::convert::TryFrom;

struct PositiveNumber(u32);

#[derive(Debug)]
struct NegativeNumberError;

impl TryFrom<i32> for PositiveNumber {
    type Error = NegativeNumberError;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 0 {
            Ok(PositiveNumber(value as u32))
        } else {
            Err(NegativeNumberError)
        }
    }
}

// Использование:
let positive = PositiveNumber::try_from(42)?;     // Ok(PositiveNumber(42))
let error = PositiveNumber::try_from(-5);         // Err(NegativeNumberError)
```

#### **Serde (для сериализации)**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// Автоматическая сериализация/десериализация JSON
let user = User {
    id: 1,
    name: "Alice".to_string(),
    email: "alice@example.com".to_string(),
};

let json = serde_json::to_string(&user)?;
let deserialized: User = serde_json::from_str(&json)?;
```

### Чек-лист по реализации типажей

Для каждого нового типа сверяйтесь со следующим списком:

```rust
#[derive(
    Debug,          // [OK] Всегда реализуйте для отладки
    Clone,          // [OK] Если тип должен поддерживать дублирование
    PartialEq,      // [OK] Если тип должен поддерживать сравнение
    Eq,             // [OK] Если сравнение рефлексивно/транзитивно
    PartialOrd,     // [OK] Если тип поддерживает упорядочение
    Ord,            // [OK] Если упорядочение является полным
    Hash,           // [OK] Если тип будет использоваться как ключ в HashMap
    Default,        // [OK] Если существует разумное значение по умолчанию
)]
struct MyType {
    // поля...
}

// Варианты ручной реализации:
impl Display for MyType { /* представление для пользователей */ }
impl From<OtherType> for MyType { /* удобное преобразование */ }
impl TryFrom<FallibleType> for MyType { /* преобразование, которое может завершиться ошибкой */ }
```

### Когда НЕ следует реализовывать типажи

- **Не реализуйте Copy для типов с данными в куче**: `String`, `Vec`, `HashMap` и т. д.
- **Не реализуйте Eq, если значения могут быть NaN**: типы, содержащие `f32`/`f64`.
- **Не реализуйте Default, если нет разумного значения по умолчанию**: дескрипторы файлов, сетевые подключения.
- **Не реализуйте Clone, если клонирование ресурсоемко**: большие структуры данных (вместо этого рассмотрите `Rc<T>`).

### Сводка: преимущества типажей

| Типаж | Преимущество | Когда использовать |
|-------|---------|-------------|
| `Debug` | `println!("{:?}", value)` | Всегда (за редкими исключениями) |
| `Display` | `println!("{}", value)` | Типы для пользовательского интерфейса/вывода |
| `Clone` | `value.clone()` | Когда явное дублирование оправдано |
| `Copy` | Неявное дублирование | Маленькие, простые типы |
| `PartialEq` | Операторы `==` и `!=` | Большинство типов |
| `Eq` | Рефлексивное равенство | Когда равенство математически обосновано |
| `PartialOrd` | `<`, `>`, `<=`, `>=` | Типы с естественным порядком |
| `Ord` | `sort()`, `BinaryHeap` | Когда упорядочение является полным |
| `Hash` | Ключи для `HashMap` | Типы, используемые в качестве ключей ассоциативного массива |
| `Default` | `Default::default()` | Типы с понятным значением по умолчанию |
| `From/Into` | Удобные преобразования | Частые преобразования типов |
| `TryFrom/TryInto` | Преобразования с обработкой ошибок | Преобразования, которые могут завершиться ошибкой |

----
