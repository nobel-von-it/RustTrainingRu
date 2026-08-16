# Трейты в Rust (Rust traits)

> **Чему вы научитесь:** Трейты — это ответ Rust на интерфейсы, абстрактные базовые классы и перегрузку операторов. Вы узнаете, как определять трейты, реализовывать их для своих типов, а также использовать динамическую диспетчеризацию (`dyn Trait`) в сравнении со статической диспетчеризацией (обобщения). Для разработчиков C++: трейты заменяют виртуальные функции, CRTP и концепты. Для разработчиков C: трейты представляют собой структурированный способ реализации полиморфизма в Rust.

- Трейты в Rust похожи на интерфейсы в других языках.
    - Трейты определяют методы, которые должны быть реализованы типами, поддерживающими этот трейт.
```rust
fn main() {
    trait Pet {
        fn speak(&self);
    }
    struct Cat;
    struct Dog;
    impl Pet for Cat {
        fn speak(&self) {
            println!("Meow");
        }
    }
    impl Pet for Dog {
        fn speak(&self) {
            println!("Woof!")
        }
    }
    let c = Cat{};
    let d = Dog{};
    c.speak();  // Между Cat и Dog нет отношения "является" (is-a)
    d.speak(); // Между Cat и Dog нет отношения "является" (is-a)
}
```

## Трейты в сравнении с концептами и интерфейсами C++

### Традиционное наследование в C++ против трейтов в Rust

```cpp
// C++ - Полиморфизм на основе наследования
class Animal {
public:
    virtual void speak() = 0;  // Чистая виртуальная функция
    virtual ~Animal() = default;
};

class Cat : public Animal {  // "Cat ЯВЛЯЕТСЯ Animal"
public:
    void speak() override {
        std::cout << "Meow" << std::endl;
    }
};

void make_sound(Animal* animal) {  // Полиморфизм времени выполнения (runtime)
    animal->speak();  // Вызов виртуальной функции
}
```

```rust
// Rust - Композиция вместо наследования с использованием трейтов
trait Animal {
    fn speak(&self);
}

struct Cat;  // Cat НЕ является Animal, но РЕАЛИЗУЕТ поведение Animal

impl Animal for Cat {  // "Cat МОЖЕТ ВЫПОЛНЯТЬ поведение Animal"
    fn speak(&self) {
        println!("Meow");
    }
}

fn make_sound<T: Animal>(animal: &T) {  // Статический полиморфизм
    animal.speak();  // Прямой вызов функции (нулевая стоимость / zero cost)
}
```

```mermaid
graph TD
    subgraph "Объектно-ориентированная иерархия C++"
        CPP_ANIMAL["Animal<br/>(Абстрактный базовый класс)"]
        CPP_CAT["Cat : public Animal<br/>(Отношение IS-A)"]
        CPP_DOG["Dog : public Animal<br/>(Отношение IS-A)"]
        
        CPP_ANIMAL --> CPP_CAT
        CPP_ANIMAL --> CPP_DOG
        
        CPP_VTABLE["Таблица виртуальных функций<br/>(Диспетчеризация во время выполнения)"]
        CPP_HEAP["Часто требует<br/>выделения памяти в куче"]
        CPP_ISSUES["[ОШИБКА] Глубокие деревья наследования<br/>[ОШИБКА] Проблема ромба<br/>[ОШИБКА] Накладные расходы времени выполнения<br/>[ОШИБКА] Жесткая связь (Tight coupling)"]
    end
    
    subgraph "Композиция на основе трейтов в Rust"
        RUST_TRAIT["trait Animal<br/>(Определение поведения)"]
        RUST_CAT["struct Cat<br/>(Только данные)"]
        RUST_DOG["struct Dog<br/>(Только данные)"]
        
        RUST_CAT -.->|"impl Animal for Cat<br/>(Поведение CAN-DO)"| RUST_TRAIT
        RUST_DOG -.->|"impl Animal for Dog<br/>(Поведение CAN-DO)"| RUST_TRAIT
        
        RUST_STATIC["Статическая диспетчеризация<br/>(Время компиляции)"]
        RUST_STACK["Возможно выделение<br/>на стеке"]
        RUST_BENEFITS["[OK] Нет иерархии наследования<br/>[OK] Несколько реализаций трейтов<br/>[OK] Нулевая стоимость времени выполнения<br/>[OK] Слабая связь (Loose coupling)"]
    end
    
    style CPP_ISSUES fill:#ff6b6b,color:#000
    style RUST_BENEFITS fill:#91e5a3,color:#000
    style CPP_VTABLE fill:#ffa07a,color:#000
    style RUST_STATIC fill:#91e5a3,color:#000
```

### Ограничения трейтов (Trait Bounds) и обобщений

```rust
use std::fmt::Display;
use std::ops::Add;

// Эквивалент шаблона C++ (меньше ограничений)
// template<typename T>
// T add_and_print(T a, T b) {
//     // Нет гарантии, что T поддерживает + или вывод
//     return a + b;  // Может дать сбой во время компиляции
// }

// Rust - явные ограничения трейтов (trait bounds)
fn add_and_print<T>(a: T, b: T) -> T 
where 
    T: Display + Add<Output = T> + Copy,
{
    println!("Adding {} + {}", a, b);  // Трейт Display
    a + b  // Трейт Add
}
```

```mermaid
graph TD
    subgraph "Эволюция ограничений обобщений"
        UNCONSTRAINED["fn process<T>(data: T)<br/>[ОШИБКА] T может быть чем угодно"]
        SINGLE_BOUND["fn process<T: Display>(data: T)<br/>[OK] T должен реализовывать Display"]
        MULTI_BOUND["fn process<T>(data: T)<br/>where T: Display + Clone + Debug<br/>[OK] Множественные требования"]
        
        UNCONSTRAINED --> SINGLE_BOUND
        SINGLE_BOUND --> MULTI_BOUND
    end
    
    subgraph "Синтаксис ограничений трейтов"
        INLINE["fn func<T: Trait>(param: T)"]
        WHERE_CLAUSE["fn func<T>(param: T)<br/>where T: Trait"]
        IMPL_PARAM["fn func(param: impl Trait)"]
        
        COMPARISON["Встроенный (Inline): Простые случаи<br/>Where: Сложные ограничения<br/>impl: Краткий синтаксис"]
    end
    
    subgraph "Магия компиляции"
        GENERIC_FUNC["Обобщенная функция<br/>с ограничениями трейтов"]
        TYPE_CHECK["Компилятор проверяет<br/>реализации трейтов"]
        MONOMORPH["Мономорфизация<br/>(Создание специализированных версий)"]
        OPTIMIZED["Полностью оптимизированный<br/>машинный код"]
        
        GENERIC_FUNC --> TYPE_CHECK
        TYPE_CHECK --> MONOMORPH
        MONOMORPH --> OPTIMIZED
        
        EXAMPLE["add_and_print::<i32><br/>add_and_print::<f64><br/>(Генерируются отдельные функции)"]
        MONOMORPH --> EXAMPLE
    end
    
    style UNCONSTRAINED fill:#ff6b6b,color:#000
    style SINGLE_BOUND fill:#ffa07a,color:#000
    style MULTI_BOUND fill:#91e5a3,color:#000
    style OPTIMIZED fill:#91e5a3,color:#000
```

### Перегрузка операторов в C++ → Трейты `std::ops` в Rust

В C++ перегрузка операторов реализуется путем написания свободных функций или функций-членов со специальными именами (`operator+`, `operator<<`, `operator[]` и т. д.). В Rust каждый оператор сопоставляется с определенным трейтом в `std::ops` (или `std::fmt` для вывода). Вместо написания функции со специальным именем вы **реализуете соответствующий трейт**.

#### Сравнение бок о бок: оператор `+`

```cpp
// C++: перегрузка оператора как метода или свободной функции
struct Vec2 {
    double x, y;
    Vec2 operator+(const Vec2& rhs) const {
        return {x + rhs.x, y + rhs.y};
    }
};

Vec2 a{1.0, 2.0}, b{3.0, 4.0};
Vec2 c = a + b;  // вызывает a.operator+(b)
```

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {
    type Output = Vec2;                     // Ассоциированный тип — результат операции +
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

let a = Vec2 { x: 1.0, y: 2.0 };
let b = Vec2 { x: 3.0, y: 4.0 };
let c = a + b;  // вызывает <Vec2 as Add>::add(a, b)
println!("{c:?}"); // Vec2 { x: 4.0, y: 6.0 }
```

#### Ключевые различия с C++

| Аспект | C++ | Rust |
|--------|-----|------|
| **Механизм** | Магические имена функций (`operator+`) | Реализация трейта (`impl Add for T`) |
| **Обнаружение** | Поиск `operator+` через grep или чтение заголовка | Просмотр реализаций трейтов — отличная поддержка в IDE |
| **Возвращаемый тип** | Свободный выбор | Фиксируется ассоциированным типом `Output` |
| **Получатель** | Обычно принимает `const T&` (заимствование) | По умолчанию принимает `self` по значению (перемещение!) |
| **Симметрия** | Можно написать `impl operator+(int, Vec2)` | Необходимо добавить `impl Add<Vec2> for i32` (применяются правила для внешних трейтов / coherence rules) |
| **`<<` для вывода** | `operator<<(ostream&, T)` — перегрузка для любого потока | `impl fmt::Display for T` — одно каноническое строковое представление (to_string) |

#### Подвох с передачей `self` по значению

В Rust метод `Add::add(self, rhs)` принимает `self` **по значению**. Для типов с трейтом `Copy` (как `Vec2` выше, который выводит `Copy`) это не проблема — компилятор копирует значения. Но для типов без `Copy` оператор `+` **потребляет (расходует)** операнды:

```rust
let s1 = String::from("hello ");
let s2 = String::from("world");
let s3 = s1 + &s2;  // s1 перемещается в s3!
// println!("{s1}");  // ❌ Ошибка компиляции: значение используется после перемещения
println!("{s2}");     // ✅ s2 был только заимствован (&s2)
```

Именно поэтому `String + &str` работает, а `&str + &str` — нет. Трейт `Add` реализован только для `String + &str`, при этом левый операнд `String` потребляется, чтобы повторно использовать его буфер. В C++ нет прямого аналога: `std::string::operator+` всегда создает новую строку.

#### Полное соответствие: Операторы C++ → Трейты Rust

| Оператор C++ | Трейт Rust | Примечания |
|-------------|-----------|-------|
| `operator+` | `std::ops::Add` | Ассоциированный тип `Output` |
| `operator-` | `std::ops::Sub` | |
| `operator*` | `std::ops::Mul` | Не разыменование указателя — для этого есть `Deref` |
| `operator/` | `std::ops::Div` | |
| `operator%` | `std::ops::Rem` | |
| `operator-` (унарный) | `std::ops::Neg` | |
| `operator!` / `operator~` | `std::ops::Not` | Rust использует `!` как для логического, так и для побитового НЕ (оператор `~` отсутствует) |
| `operator&`, `\|`, `^` | `BitAnd`, `BitOr`, `BitXor` | |
| `operator<<`, `>>` (сдвиг) | `Shl`, `Shr` | НЕ для потокового ввода-вывода! |
| `operator+=` | `std::ops::AddAssign` | Принимает `&mut self` (а не `self`) |
| `operator[]` | `std::ops::Index` / `IndexMut` | Возвращает `&Output` / `&mut Output` |
| `operator()` | `Fn` / `FnMut` / `FnOnce` | Реализуются замыканиями; вы не можете реализовать `Fn` напрямую |
| `operator==` | `PartialEq` (+ `Eq`) | В `std::cmp`, а не в `std::ops` |
| `operator<` | `PartialOrd` (+ `Ord`) | В `std::cmp` |
| `operator<<` (поток) | `fmt::Display` | `println!("{}", x)` |
| `operator<<` (отладка) | `fmt::Debug` | `println!("{:?}", x)` |
| `operator bool` | Нет прямого эквивалента | Используйте `impl From<T> for bool` или именованный метод вроде `.is_empty()` |
| `operator T()` (неявное приведение) | Нет неявных преобразований | Используйте трейты `From`/`Into` (явно) |

#### Защитные барьеры: что предотвращает Rust

1. **Нет неявных преобразований**: В C++ `operator int()` может приводить к скрытым и неожиданным приведениям типов. В Rust нет операторов неявного преобразования — используйте `From`/`Into` и вызывайте метод `.into()` явно.
2. **Нет перегрузки `&&` / `||`**: C++ разрешает это (что нарушает семантику короткого замыкания!). Rust — нет.
3. **Нет перегрузки `=`**: Присваивание в Rust — это всегда перемещение или копирование, его логику нельзя переопределить. Составное присваивание (`+=`) МОЖНО перегрузить с помощью `AddAssign` и других аналогичных трейтов.
4. **Нет перегрузки оператора запятой `,`**: C++ позволяет перегружать `operator,()`, что является одним из самых известных источников ошибок в C++. Rust этого не позволяет.
5. **Нет перегрузки `&` (взятие адреса)**: Еще один источник проблем в C++ (для обхода которого существует `std::addressof`). В Rust символ `&` всегда означает «заимствование».
6. **Правила когерентности (Coherence rules)**: Вы можете реализовать только `Add<Foreign>` для вашего собственного типа или `Add<YourType>` для стороннего типа, но никогда — `Add<Foreign>` для стороннего типа (`Foreign`). Это предотвращает конфликты определений операторов между различными библиотеками (крейтами).

> **Итог**: В C++ перегрузка операторов мощна, но практически не регулируется — можно перегрузить почти всё, включая запятую и взятие адреса, а неявные преобразования могут происходить незаметно. Rust предоставляет аналогичную выразительность для арифметических операторов и операторов сравнения через трейты, но **блокирует исторически опасные перегрузки** и требует, чтобы все преобразования были явными.

----
# Трейты в Rust

- Rust позволяет реализовывать пользовательский трейт даже для встроенных типов (таких как u32 в этом примере). Однако либо трейт, либо сам тип должны принадлежать текущему крейту.
```rust
trait IsSecret {
  fn is_secret(&self);
}
// Трейт IsSecret принадлежит текущему крейту, поэтому всё в порядке
impl IsSecret for u32 {
  fn is_secret(&self) {
      if *self == 42 {
          println!("Is secret of life");
      }
  }
}

fn main() {
  42u32.is_secret();
  43u32.is_secret();
}
```


# Трейты в Rust
- Трейты поддерживают наследование интерфейсов и реализации по умолчанию
```rust
trait Animal {
  // Реализация по умолчанию
  fn is_mammal(&self) -> bool {
    true
  }
}
trait Feline : Animal {
  // Реализация по умолчанию
  fn is_feline(&self) -> bool {
    true
  }
}

struct Cat;
// Использование реализаций по умолчанию. Обратите внимание, что все супертрейты должны быть реализованы индивидуально
impl Feline for Cat {}
impl Animal for Cat {}
fn main() {
  let c = Cat{};
  println!("{} {}", c.is_mammal(), c.is_feline());
}
```
----
# Упражнение: Реализация трейта Logger

🟡 **Средний уровень**

- Реализуйте трейт `Log` с единственным методом `log()`, который принимает `u64`
    - Реализуйте два различных логгера `SimpleLogger` и `ComplexLogger`, реализующих трейт `Log`. Один должен выводить "Simple logger" вместе с `u64`, а другой — "Complex logger" вместе с `u64`. 

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
trait Log {
    fn log(&self, value: u64);
}

struct SimpleLogger;
struct ComplexLogger;

impl Log for SimpleLogger {
    fn log(&self, value: u64) {
        println!("Simple logger: {value}");
    }
}

impl Log for ComplexLogger {
    fn log(&self, value: u64) {
        println!("Complex logger: {value} (hex: 0x{value:x}, binary: {value:b})");
    }
}

fn main() {
    let simple = SimpleLogger;
    let complex = ComplexLogger;
    simple.log(42);
    complex.log(42);
}
// Вывод:
// Simple logger: 42
// Complex logger: 42 (hex: 0x2a, binary: 101010)
```

</details>

----
# Ассоциированные типы в трейтах Rust
```rust
#[derive(Debug)]
struct Small(u32);
#[derive(Debug)]
struct Big(u32);
trait Double {
    type T;
    fn double(&self) -> Self::T;
}

impl Double for Small {
    type T = Big;
    fn double(&self) -> Self::T {
        Big(self.0 * 2)
    }
}
fn main() {
    let a = Small(42);
    println!("{:?}", a.double());
}
```

# Использование impl с трейтами в Rust
- Ключевое слово `impl` может использоваться с трейтами для принятия любого типа, реализующего данный трейт
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &impl Pet) {
    p.speak();
}
fn main() {
    let c = Cat {};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```

# Использование impl с трейтами в Rust
- `impl` также может использоваться в качестве возвращаемого значения
```rust
trait Pet {}
struct Dog;
struct Cat;
impl Pet for Cat {}
impl Pet for Dog {}
fn cat_as_pet() -> impl Pet {
    let c = Cat {};
    c
}
fn dog_as_pet() -> impl Pet {
    let d = Dog {};
    d
}
fn main() {
    let p = cat_as_pet();
    let d = dog_as_pet();
}
```
----
# Динамические трейты в Rust
- Динамические трейты позволяют вызывать методы трейта без знания базового типа. Это поведение называется стиранием типов (type erasure)
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {x: u32}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &dyn Pet) {
    p.speak();
}
fn main() {
    let c = Cat {x: 42};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```
----

## Выбор между `impl Trait`, `dyn Trait` и `enum`

Все три подхода позволяют достичь полиморфизма, но имеют разные компромиссы:

| Подход | Диспетчеризация | Производительность | Гетерогенные коллекции? | Когда использовать |
|----------|----------|-------------|---------------------------|-------------|
| `impl Trait` / обобщения | Статическая (мономорфизация) | Нулевая стоимость — встраивается во время компиляции | Нет — каждый элемент имеет один конкретный тип | Выбор по умолчанию. Аргументы функций, возвращаемые типы |
| `dyn Trait` | Динамическая (vtable) | Небольшие накладные расходы на вызов (~1 косвенное обращение по указателю) | Да — `Vec<Box<dyn Trait>>` | Когда вам нужны разные типы в одной коллекции или расширяемость в стиле плагинов |
| `enum` | Сопоставление шаблонов (Match) | Нулевая стоимость — варианты известны во время компиляции | Да — но только для известных вариантов | Когда набор вариантов фиксирован (закрыт) и известен во время компиляции |

```rust
trait Shape {
    fn area(&self) -> f64;
}
struct Circle { radius: f64 }
struct Rect { w: f64, h: h64 }
impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius } }
impl Shape for Rect   { fn area(&self) -> f64 { self.w * self.h } }

// Статическая диспетчеризация — компилятор генерирует отдельный код для каждого типа
fn print_area(s: &impl Shape) { println!("{}", s.area()); }

// Динамическая диспетчеризация — одна функция, работает с любой фигурой за указателем
fn print_area_dyn(s: &dyn Shape) { println!("{}", s.area()); }

// Enum — закрытый набор вариантов, трейт не требуется
enum ShapeEnum { Circle(f64), Rect(f64, f64) }
impl ShapeEnum {
    fn area(&self) -> f64 {
        match self {
            ShapeEnum::Circle(r) => std::f64::consts::PI * r * r,
            ShapeEnum::Rect(w, h) => w * h,
        }
    }
}
```

> **Для разработчиков C++:** `impl Trait` аналогичен шаблонам C++ (мономорфизированный, с нулевой стоимостью). `dyn Trait` аналогичен виртуальным функциям C++ (диспетчеризация через vtable). Перечисления Rust с сопоставлением `match` похожи на `std::variant` с `std::visit`, но компилятор строго требует исчерпывающего перебора вариантов.

> **Практическое правило**: Начинайте с `impl Trait` (статическая диспетчеризация). Переходите к `dyn Trait` только тогда, когда вам нужны гетерогенные коллекции или когда конкретный тип неизвестен на этапе компиляции. Используйте `enum`, если вы полностью контролируете все возможные варианты.
