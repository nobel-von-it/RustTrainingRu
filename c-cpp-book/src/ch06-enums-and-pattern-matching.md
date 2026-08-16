# Типы перечислений в Rust

> **Что вы узнаете:** Перечисления в Rust как размеченные объединения (правильно реализованные tagged unions), сопоставление `match` для исчерпывающего поиска по образцу, а также то, как перечисления заменяют иерархию классов C++ и размеченные объединения C с гарантированной компилятором безопасностью.

- Типы перечислений (enum) представляют собой размеченные объединения (discriminated unions), то есть они являются суммой нескольких возможных различных типов с тегом, идентифицирующим конкретный вариант
    - Для разработчиков C: перечисления в Rust могут содержать данные (правильно реализованные размеченные объединения — компилятор сам отслеживает, какой вариант активен)
    - Для разработчиков C++: перечисления в Rust похожи на `std::variant`, но с исчерпывающим сопоставлением с образцом, отсутствием исключений `std::get` и без шаблонного кода (boilerplate) для `std::visit`
    - Размер `enum` равен размеру наибольшего из его возможных типов. Отдельные варианты никак не связаны друг с другом и могут иметь совершенно разные типы данных
    - Типы `enum` — одна из самых мощных возможностей языка: они заменяют собой целые иерархии классов в C++ (подробнее об этом см. в разделе практических примеров)
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let a = Numbers::Zero;
    let b = Numbers::SmallNumber(42);
    let c : Numbers = a; // Отлично — тип переменной a выведен как Numbers
    let d : Numbers = b; // Отлично — тип переменной b выведен как Numbers
}
```
----
# Конструкция match в Rust
- Конструкция ```match``` в Rust — это аналог "switch" в языке C на стероидах
    - ```match``` может использоваться для сопоставления с образцом для простых типов данных, ```struct```, ```enum```
    - Конструкция ```match``` должна быть исчерпывающей (exhaustive), то есть должны быть покрыты все возможные случаи для данного типа. Символ подчеркивания ```_``` используется в качестве шаблона для случая «все остальное»
    - ```match``` может возвращать значение, но все ветви (```=>```) должны возвращать значение одного и того же типа

```rust
fn main() {
    let x = 42;
    // В данном случае _ покрывает все числа, кроме явно перечисленных
    let is_secret_of_life = match x {
        42 => true, // возвращаемый тип — булево значение
        _ => false, // возвращаемый тип — булево значение
        // Это не скомпилируется, так как возвращаемый тип не bool
        // _ => 0  
    };
    println!("{is_secret_of_life}");
}
```

# Конструкция match в Rust (продолжение)
- ```match``` поддерживает диапазоны, булевы фильтры и защитные условия ```if``` (match guards)
```rust
fn main() {
    let x = 42;
    match x {
        // Обратите внимание, что запись =41 задает инклюзивный диапазон
        0..=41 => println!("Less than the secret of life"),
        42 => println!("Secret of life"),
        _ => println!("More than the secret of life"),
    }
    let y = 100;
    match y {
        100 if x == 43 => println!("y is 100% not secret of life"),
        100 if x == 42 => println!("y is 100% secret of life"),
        _ => (),    // Ничего не делать
    }
}
```

# Конструкция match в Rust (продолжение)
- ```match``` и ```enum``` часто используются вместе
    - Конструкция match может «связывать» содержащееся значение с переменной. Используйте ```_```, если значение не важно
    - Макрос ```matches!``` может использоваться для проверки на совпадение с конкретным вариантом
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let b = Numbers::SmallNumber(42);
    match b {
        Numbers::Zero => println!("Zero"),
        Numbers::SmallNumber(value) => println!("Small number {value}"),
        Numbers::BiggerNumber(_) | Numbers::EvenBiggerNumber(_) => println!("Some BiggerNumber or EvenBiggerNumber"),
    }
    
    // Булева проверка для определенных вариантов
    if matches!(b, Numbers::Zero | Numbers::SmallNumber(_)) {
        println!("Matched Zero or small number");
    }
}
```

# Конструкция match в Rust (продолжение)
- ```match``` также позволяет сопоставлять данные с использованием деструктуризации и срезов
```rust
fn main() {
    struct Foo {
        x: (u32, bool),
        y: u32
    }
    let f = Foo {x: (42, true), y: 100};
    match f {
        // Захватывает значение поля x в переменную с именем tuple
        Foo{y: 100, x : tuple} => println!("Matched x: {tuple:?}"),
        _ => ()
    }
    let a = [40, 41, 42];
    match a {
        // Последним элементом среза должно быть 42. Символ @ используется для связывания совпадения
        [rest @ .., 42] => println!("{rest:?}"),
        // Первым элементом среза должно быть 42. Символ @ используется для связывания совпадения
        [42, rest @ ..] => println!("{rest:?}"),
        _ => (),
    }
}
```

# Упражнение: Реализация сложения и вычитания с использованием match и enum

🟢 **Starter**

- Напишите функцию, которая реализует арифметические операции над 64-разрядными беззнаковыми числами
- **Шаг 1**: Определите перечисление для операций:
```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}
```
- **Шаг 2**: Определите перечисление результата:
```rust
enum CalcResult {
    Ok(u64),                    // Успешный результат
    Invalid(String),            // Сообщение об ошибке для невалидных операций
}
```
- **Шаг 3**: Реализуйте функцию `calculate(op: Operation) -> CalcResult`
    - Для Add: возвращает Ok(sum)
    - Для Subtract: возвращает Ok(difference), если первый операнд больше либо равен второму, иначе — Invalid("Underflow")
- **Подсказка**: Используйте сопоставление с образцом в вашей функции:
```rust
match op {
    Operation::Add(a, b) => { /* ваш код */ },
    Operation::Subtract(a, b) => { /* ваш код */ },
}
```

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}

enum CalcResult {
    Ok(u64),
    Invalid(String),
}

fn calculate(op: Operation) -> CalcResult {
    match op {
        Operation::Add(a, b) => CalcResult::Ok(a + b),
        Operation::Subtract(a, b) => {
            if a >= b {
                CalcResult::Ok(a - b)
            } else {
                CalcResult::Invalid("Underflow".to_string())
            }
        }
    }
}

fn main() {
    match calculate(Operation::Add(10, 20)) {
        CalcResult::Ok(result) => println!("10 + 20 = {result}"),
        CalcResult::Invalid(msg) => println!("Error: {msg}"),
    }
    match calculate(Operation::Subtract(5, 10)) {
        CalcResult::Ok(result) => println!("5 - 10 = {result}"),
        CalcResult::Invalid(msg) => println!("Error: {msg}"),
    }
}
// Вывод:
// 10 + 20 = 30
// Error: Underflow
```

</details>

# Связанные методы в Rust
- С помощью ```impl``` можно определять связанные методы (associated methods) для таких типов, как ```struct```, ```enum``` и т. д.
    - Эти методы могут опционально принимать ```self``` в качестве параметра. Передача ```self``` концептуально похожа на передачу указателя на структуру первым параметром в C или использование ```this``` в C++
    - Ссылка на ```self``` может быть неизменяемой (по умолчанию: ```&self```), изменяемой (```&mut self```), или принимать значение целиком (```self``` — с передачей владения)
    - Ключевое слово ```Self``` может использоваться как сокращение для обозначения самого типа
```rust
struct Point {x: u32, y: u32}
impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point {x, y}
    }
    fn increment_x(&mut self) {
        self.x += 1;
    }
}
fn main() {
    let mut p = Point::new(10, 20);
    p.increment_x();
}
```

# Упражнение: Сложение и преобразование точек (Point)

🟡 **Средний** — требует понимания разницы между перемещением (move) и заимствованием (borrow) на основе сигнатур методов
- Реализуйте следующие связанные методы для ```Point```
    - ```add()``` будет принимать другую точку ```Point``` и увеличивать значения x и y по месту (подсказка: используйте ```&mut self```)
    - ```transform()``` будет поглощать существующую точку ```Point``` (подсказка: используйте ```self```) и возвращать новую точку ```Point``` со значениями x и y, возведенными в квадрат

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
struct Point { x: u32, y: u32 }

impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point { x, y }
    }
    fn add(&mut self, other: &Point) {
        self.x += other.x;
        self.y += other.y;
    }
    fn transform(self) -> Point {
        Point { x: self.x * self.x, y: self.y * self.y }
    }
}

fn main() {
    let mut p1 = Point::new(2, 3);
    let p2 = Point::new(10, 20);
    p1.add(&p2);
    println!("After add: x={}, y={}", p1.x, p1.y);           // x=12, y=23
    let p3 = p1.transform();
    println!("After transform: x={}, y={}", p3.x, p3.y);     // x=144, y=529
    // p1 больше недоступна — метод transform() поглотил ее
}
```

</details>

----
