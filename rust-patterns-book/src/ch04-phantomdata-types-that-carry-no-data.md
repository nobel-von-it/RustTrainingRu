# 4. PhantomData — типы, не содержащие данных 🔴

> **Чему вы научитесь:**
> - Почему существует `PhantomData<T>` и какие три проблемы он решает
> - Брендированию времени жизни (lifetime branding) для принудительного контроля областей видимости на этапе компиляции
> - Шаблону единиц измерения (unit-of-measure) для вычислений, безопасных с точки зрения размерностей
> - Вариантности (ковариантность, контравариантность, инвариантность) и тому, как PhantomData управляет ею

## Какие задачи решает PhantomData

`PhantomData<T>` — это тип нулевого размера, который сообщает компилятору: «эта структура логически связана с `T`, даже если она не содержит `T`». Он влияет на вариантность (variance), проверку освобождения памяти (drop check) и автоматический вывод трейтов (auto-trait inference) — не потребляя при этом памяти.

```rust
use std::marker::PhantomData;

// Без PhantomData:
struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
    // Проблема: компилятор не знает, что эта структура заимствует из 'a,
    // или что она связана с T для целей проверки освобождения (drop check)
}

// С PhantomData:
struct Slice<'a, T> {
    ptr: *const T,
    len: usize,
    _marker: PhantomData<&'a T>,
    // Теперь компилятор знает, что:
    // 1. Эта структура заимствует данные с временем жизни 'a
    // 2. Она ковариантна по 'a (время жизни может уменьшаться)
    // 3. Проверка освобождения (drop check) учитывает T
}
```

**Три задачи PhantomData**:

| Задача | Пример | Что делает |
|-----|---------|-------------|
| **Связывание времени жизни** | `PhantomData<&'a T>` | Структура обрабатывается как заимствующая `'a` |
| **Симуляция владения** | `PhantomData<T>` | Проверка освобождения (drop check) предполагает, что структура владеет `T` |
| **Контроль вариантности** | `PhantomData<fn(T)>` | Делает структуру контравариантной по отношению к `T` |

### Брендирование времени жизни (Lifetime Branding)

Используйте `PhantomData`, чтобы предотвратить смешивание значений из разных «сессий» или «контекстов»:

```rust
use std::cell::RefCell;
use std::marker::PhantomData;

/// Дескриптор, валидный только в течение времени жизни определенной арены
struct ArenaHandle<'arena> {
    index: usize,
    _brand: PhantomData<&'arena ()>,
}

struct Arena {
    data: RefCell<Vec<String>>,
}

impl Arena {
    fn new() -> Self {
        Arena { data: RefCell::new(Vec::new()) }
    }

    /// Выделить память под строку и вернуть брендированный дескриптор
    fn alloc(&self, value: String) -> ArenaHandle<'_> {
        let mut data = self.data.borrow_mut();
        let index = data.len();
        data.push(value);
        ArenaHandle { index, _brand: PhantomData }
    }

    /// Поиск по дескриптору — принимает только дескрипторы из ЭТОЙ арены
    fn get<'a>(&'a self, handle: ArenaHandle<'a>) -> String {
        let data = self.data.borrow();
        data[handle.index].clone()
    }
}

fn main() {
    let arena1 = Arena::new();
    let handle1 = arena1.alloc("hello".to_string());

    // Нельзя использовать handle1 с другой ареной — времена жизни не совпадут
    // let arena2 = Arena::new();
    // arena2.get(handle1); // ❌ Несовпадение времени жизни (lifetime mismatch)

    println!("{}", arena1.get(handle1)); // ✅
}
```

### Шаблон единиц измерения (Unit-of-Measure)

Предотвратите смешивание несовместимых единиц измерения на этапе компиляции с нулевыми накладными расходами времени выполнения:

```rust
use std::marker::PhantomData;
use std::ops::{Add, Mul};

// Типы-маркеры единиц измерения (нулевого размера)
struct Meters;
struct Seconds;
struct MetersPerSecond;

#[derive(Debug, Clone, Copy)]
struct Quantity<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<U> Quantity<U> {
    fn new(value: f64) -> Self {
        Quantity { value, _unit: PhantomData }
    }
}

// Можно складывать только одинаковые единицы измерения:
impl<U> Add for Quantity<U> {
    type Output = Quantity<U>;
    fn add(self, rhs: Self) -> Self::Output {
        Quantity::new(self.value + rhs.value)
    }
}

// Meters / Seconds = MetersPerSecond (пользовательский трейт)
impl std::ops::Div<Quantity<Seconds>> for Quantity<Meters> {
    type Output = Quantity<MetersPerSecond>;
    fn div(self, rhs: Quantity<Seconds>) -> Quantity<MetersPerSecond> {
        Quantity::new(self.value / rhs.value)
    }
}

fn main() {
    let dist = Quantity::<Meters>::new(100.0);
    let time = Quantity::<Seconds>::new(9.58);
    let speed = dist / time; // Quantity<MetersPerSecond>
    println!("Speed: {:.2} m/s", speed.value); // 10.44 m/s

    // let nonsense = dist + time; // ❌ Ошибка компиляции: нельзя сложить Meters + Seconds
}
```

> **Это чистая магия системы типов** — `PhantomData<Meters>` имеет нулевой размер, поэтому `Quantity<Meters>` располагается в памяти точно так же, как `f64`. Никаких накладных расходов на обёртку во время выполнения, но полная безопасность единиц измерения на этапе компиляции.

### PhantomData и проверка освобождения (Drop Check)

Когда компилятор проверяет, может ли деструктор структуры получить доступ к устаревшим/недействительным данным, он использует `PhantomData` для принятия решения:

```rust
use std::marker::PhantomData;

// PhantomData<T> — компилятор предполагает, что мы МОЖЕМ удалить T
// Это означает, что T должен пережить нашу структуру
struct OwningSemantic<T> {
    ptr: *const T,
    _marker: PhantomData<T>,  // «Я логически владею T»
}

// PhantomData<*const T> — компилятор предполагает, что мы НЕ владеем T
// Более лояльный вариант — T не обязан переживать нас
struct NonOwningSemantic<T> {
    ptr: *const T,
    _marker: PhantomData<*const T>,  // «Я просто указываю на T»
}
```

**Практическое правило**: При оборачивании сырых указателей (raw pointers) тщательно выбирайте PhantomData:
- Пишете контейнер, владеющий своими данными? → `PhantomData<T>`
- Пишете представление (view) или ссылочный тип? → `PhantomData<&'a T>` или `PhantomData<*const T>`

### Вариантность — почему важен параметр типа PhantomData

**Вариантность (variance)** определяет, может ли обобщённый тип быть заменен на подтип или супертип (в Rust «подтип» означает «имеет более длительное время жизни»). Ошибка в определении вариантности приводит либо к отклонению корректного кода, либо к компиляции небезопасного кода.

```mermaid
graph LR
    subgraph Ковариантный (Covariant)
        direction TB
        A1["&'long T"] -->|"может стать"| A2["&'short T"]
    end

    subgraph Контравариантный (Contravariant)
        direction TB
        B1["fn(&'short T)"] -->|"может стать"| B2["fn(&'long T)"]
    end

    subgraph Инвариантный (Invariant)
        direction TB
        C1["&'a mut T"] ---|"подстановка НЕВОЗМОЖНА"| C2["&'b mut T"]
    end

    style A1 fill:#d4efdf,stroke:#27ae60,color:#000
    style A2 fill:#d4efdf,stroke:#27ae60,color:#000
    style B1 fill:#e8daef,stroke:#8e44ad,color:#000
    style B2 fill:#e8daef,stroke:#8e44ad,color:#000
    style C1 fill:#fadbd8,stroke:#e74c3c,color:#000
    style C2 fill:#fadbd8,stroke:#e74c3c,color:#000
```

#### Три вида вариантности

| Вариантность | Смысл | «Могу ли я подставить...» | Пример в Rust |
|----------|---------|---------------------|--------------|
| **Ковариантность** | Подтип наследует направление | `'long` там, где ожидается `'short` ✅ | `&'a T`, `Vec<T>`, `Box<T>` |
| **Контравариантность** | Подтип идет *против* направления | `'short` там, где ожидается `'long` ✅ | `fn(T)` (в позиции аргумента) |
| **Инвариантность** | Подстановка не допускается | Ни в одну из сторон ✅ | `&mut T`, `Cell<T>`, `UnsafeCell<T>` |

#### Почему `&'a T` ковариантен по `'a`

```rust
fn print_str(s: &str) {
    println!("{s}");
}

fn main() {
    let owned = String::from("hello");
    // owned живет в течение всей функции ('long)
    // print_str ожидает &'_ str ('short — только на время вызова)
    print_str(&owned); // ✅ Ковариантность: переход 'long → 'short безопасен
    // Ссылку с большим временем жизни всегда можно использовать там, где требуется меньшее время жизни.
}
```

#### Почему `&mut T` инвариантен по `T`

```rust
// Если бы &mut T был ковариантен по T, то следующий код скомпилировался бы:
fn evil(s: &mut &'static str) {
    // Мы могли бы записать ссылку &str с меньшим временем жизни в ячейку &'static str!
    let local = String::from("temporary");
    // *s = &local; // ← Это привело бы к появлению висячей ссылки &'static str
}

// Инвариантность предотвращает это: &'static str ≠ &'a str при мутации.
// Компилятор полностью отклоняет такую подстановку.
```

#### Как PhantomData служит для контроля вариантности

`PhantomData<X>` наделяет вашу структуру **такой же вариантностью, как у `X`**:

```rust
use std::marker::PhantomData;

// Ковариантен по 'a — Ref<'long> может быть использован как Ref<'short>
struct Ref<'a, T> {
    ptr: *const T,
    _marker: PhantomData<&'a T>,  // Ковариантен по 'a, ковариантен по T
}

// Инвариантен по T — предотвращает небезопасное сокращение времени жизни T
struct MutRef<'a, T> {
    ptr: *mut T,
    _marker: PhantomData<&'a mut T>,  // Ковариантен по 'a, ИНВАРИАНТЕН по T
}

// Контравариантен по T — полезно для контейнеров обратного вызова (callbacks)
struct CallbackSlot<T> {
    _marker: PhantomData<fn(T)>,  // Контравариантен по T
}
```

**Шпаргалка по вариантности PhantomData**:

| Тип PhantomData | Вариантность по `T` | Вариантность по `'a` | Когда использовать |
|------------------|--------------------|--------------------|-----------|
| `PhantomData<T>` | Ковариантный | — | Вы логически владеете `T` |
| `PhantomData<&'a T>` | Ковариантный | Ковариантный | Вы заимствуете `T` с временем жизни `'a` |
| `PhantomData<&'a mut T>` | **Инвариантный** | Ковариантный | Вы мутабельно заимствуете `T` |
| `PhantomData<*const T>` | Ковариантный | — | Невладеющий указатель на `T` |
| `PhantomData<*mut T>` | **Инвариантный** | — | Невладеющий мутабельный указатель |
| `PhantomData<fn(T)>` | **Контравариантный** | — | `T` находится в позиции аргумента |
| `PhantomData<fn() -> T>` | Ковариантный | — | `T` находится в позиции возвращаемого значения |
| `PhantomData<fn(T) -> T>` | **Инвариантный** | — | `T` в обеих позициях взаимно компенсируется |

#### Подробный пример: почему это важно на практике

```rust
use std::marker::PhantomData;

// Токен, который брендирует значения временем жизни сессии.
// ДОЛЖЕН быть ковариантным по 'a — иначе вызывающий код не сможет сократить
// время жизни при передаче в функции, которым требуется меньшее время жизни заимствования.
struct SessionToken<'a> {
    id: u64,
    _brand: PhantomData<&'a ()>,  // ✅ Ковариантно — вызывающие стороны могут сокращать 'a
    // _brand: PhantomData<fn(&'a ())>,  // ❌ Контравариантно — нарушает эргономику использования
    // _brand: PhantomData<&'a mut ()>;  // Все еще ковариантно по 'a (инвариантно по T, но T фиксирован как ())
}

fn use_token(token: &SessionToken<'_>) {
    println!("Использование токена {}", token.id);
}

fn main() {
    let token = SessionToken { id: 42, _brand: PhantomData };
    use_token(&token); // ✅ Работает, так как SessionToken ковариантен по 'a
}
```

> **Правило выбора**: Начинайте с `PhantomData<&'a T>` (ковариантный). Переходите к
> `PhantomData<&'a mut T>` (инвариантный) только в том случае, если ваша абстракция предоставляет
> мутабельный доступ к `T`. Практически никогда не используйте `PhantomData<fn(T)>` (контравариантный) —
> он корректен только для сценариев хранения обратных вызовов (callback-storage).

> **Основные выводы — PhantomData**
> - `PhantomData<T>` переносит информацию о типе и времени жизни без накладных расходов во время выполнения
> - Используйте его для брендирования времени жизни, контроля вариантности и шаблонов единиц измерения
> - Проверка освобождения (drop check): `PhantomData<T>` сообщает компилятору, что ваш тип логически владеет `T`

> **См. также:** [Гл. 3 — Newtype и Type-State](ch03-the-newtype-and-type-state-patterns.md) для изучения шаблонов type-state, использующих PhantomData. [Гл. 11 — Небезопасный Rust](ch12-unsafe-rust-controlled-danger.md) для понимания взаимодействия PhantomData с сырыми указателями.

---

### Упражнение: Единицы измерения с PhantomData ★★ (~30 мин)

Расширьте шаблон единиц измерения для поддержки следующих возможностей:
- `Meters`, `Seconds`, `Kilograms` (Метры, Секунды, Килограммы)
- Сложение одинаковых единиц измерения
- Умножение: `Meters * Meters = SquareMeters` (Квадратные метры)
- Деление: `Meters / Seconds = MetersPerSecond` (Метры в секунду)

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
    println!("Площадь: {:.1} м²", area.value);

    let dist = Qty::<Meters>::new(100.0);
    let time = Qty::<Seconds>::new(9.58);
    let speed = dist / time;
    println!("Скорость: {:.2} м/с", speed.value);

    let sum = width + height; // Одинаковые единицы ✅
    println!("Сумма: {:.1} м", sum.value);

    // let bad = width + time; // ❌ Ошибка компиляции: нельзя сложить Meters + Seconds
}
```

</details>

***
