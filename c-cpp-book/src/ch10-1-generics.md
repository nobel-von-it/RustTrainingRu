# Обобщенные типы в Rust (Generics)

> **Чему вы научитесь:** Параметры обобщенных типов (Generics), мономорфизация (обобщения с нулевой стоимостью), ограничения трейтов (trait bounds) и сравнение обобщений Rust с шаблонами C++ — с более понятными сообщениями об ошибках и без необходимости в SFINAE.

- Обобщения (Generics) позволяют повторно использовать один и тот же алгоритм или структуру данных для различных типов данных.
    - Параметр обобщения записывается как идентификатор внутри угловых скобок `<>`, например: `<T>`. Параметр может иметь любое допустимое имя идентификатора, но для краткости обычно используют одну букву.
    - Компилятор выполняет мономорфизацию на этапе компиляции, то есть генерирует новый тип для каждого встреченного варианта `T`.
```rust
// Возвращает кортеж типа <T>, состоящий из левого и правого элементов типа <T>
fn pick<T>(x: u32, left: T, right: T) -> (T, T) {
   if x == 42 {
    (left, right) 
   } else {
    (right, left)
   }
}
fn main() {
    let a = pick(42, true, false);
    let b = pick(42, "hello", "world");
    println!("{a:?}, {b:?}");
}
```

# Обобщенные типы в Rust
- Обобщения также могут применяться к типам данных и связанным с ними методам. Можно специализировать реализацию для конкретного `<T>` (например, для `f32` в отличие от `u32`).
```rust
#[derive(Debug)] // Мы обсудим это позже
struct Point<T> {
    x : T,
    y : T,
}
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point {x, y}
    }
    fn set_x(&mut self, x: T) {
         self.x = x;       
    }
    fn set_y(&mut self, y: T) {
         self.y = y;       
    }
}
impl Point<f32> {
    fn is_secret(&self) -> bool {
        self.x == 42.0
    }    
}
fn main() {
    let mut p = Point::new(2, 4); // i32
    let q = Point::new(2.0, 4.0); // f32
    p.set_x(42);
    p.set_y(43);
    println!("{p:?} {q:?} {}", q.is_secret());
}
```

# Упражнение: Обобщения

🟢 **Начальный уровень**
- Измените тип `Point` так, чтобы для `x` и `y` использовались два разных типа (`T` и `U`).

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
#[derive(Debug)]
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn new(x: T, y: U) -> Self {
        Point { x, y }
    }
}

fn main() {
    let p1 = Point::new(42, 3.14);        // Point<i32, f64>
    let p2 = Point::new("hello", true);   // Point<&str, bool>
    let p3 = Point::new(1u8, 1000u64);    // Point<u8, u64>
    println!("{p1:?}");
    println!("{p2:?}");
    println!("{p3:?}");
}
// Вывод:
// Point { x: 42, y: 3.14 }
// Point { x: "hello", y: true }
// Point { x: 1, y: 1000 }
```

</details>

### Сочетание трейтов и обобщений в Rust
- Трейты можно использовать для наложения ограничений на обобщенные типы.
- Ограничение может быть указано с помощью двоеточия `:` после параметра обобщенного типа или с помощью ключевого слова `where`. В следующем примере определяется обобщенная функция `get_area`, которая принимает любой тип `T`, при условии, что он реализует трейт `ComputeArea`.
```rust
    trait ComputeArea {
        fn area(&self) -> u64;
    }
    fn get_area<T: ComputeArea>(t: &T) -> u64 {
        t.area()
    }
```
- [▶ Попробовать в Rust Playground](https://play.rust-lang.org/)

### Сочетание трейтов и обобщений in Rust
- Можно использовать несколько ограничений трейтов одновременно.
```rust
trait Fish {}
trait Mammal {}
struct Shark;
struct Whale;
impl Fish for Shark {}
impl Fish for Whale {}
impl Mammal for Whale {}
fn only_fish_and_mammals<T: Fish + Mammal>(_t: &T) {}
fn main() {
    let w = Whale {};
    only_fish_and_mammals(&w);
    let _s = Shark {};
    // Не скомпилируется
    only_fish_and_mammals(&_s);
}
```

### Ограничения трейтов в типах данных Rust
- Ограничения трейтов можно сочетать с обобщениями в типах данных.
- В следующем примере мы определяем трейт `PrintDescription` и обобщенную структуру `Shape` с полем, ограниченным этим трейтом.
```rust
trait PrintDescription {
    fn print_description(&self);
}
struct Shape<S: PrintDescription> {
    shape: S,
}
// Обобщенная реализация Shape для любого типа, реализующего PrintDescription
impl<S: PrintDescription> Shape<S> {
    fn print(&self) {
        self.shape.print_description();
    }
}
```
- [▶ Попробовать в Rust Playground](https://play.rust-lang.org/)

# Упражнение: Ограничения трейтов и обобщения

🟡 **Средний уровень**
- Реализуйте структуру с обобщенным полем `cipher`, который реализует `CipherText`.
```rust
trait CipherText {
    fn encrypt(&self);
}
// TO DO
//struct Cipher<>

```
- Затем реализуйте метод `encrypt` в блоке `impl` структуры, который вызывает метод `encrypt` у поля `cipher`.
```rust
// TO DO
impl for Cipher<> {}
```
- Далее реализуйте трейт `CipherText` для двух структур `CipherOne` и `CipherTwo` (достаточно просто вызвать `println()`). Создайте экземпляры `CipherOne` и `CipherTwo` и используйте `Cipher` для их вызова.

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
trait CipherText {
    fn encrypt(&self);
}

struct Cipher<T: CipherText> {
    cipher: T,
}

impl<T: CipherText> Cipher<T> {
    fn encrypt(&self) {
        self.cipher.encrypt();
    }
}

struct CipherOne;
struct CipherTwo;

impl CipherText for CipherOne {
    fn encrypt(&self) {
        println!("CipherOne encryption applied");
    }
}

impl CipherText for CipherTwo {
    fn encrypt(&self) {
        println!("CipherTwo encryption applied");
    }
}

fn main() {
    let c1 = Cipher { cipher: CipherOne };
    let c2 = Cipher { cipher: CipherTwo };
    c1.encrypt();
    c2.encrypt();
}
// Вывод:
// CipherOne encryption applied
// CipherTwo encryption applied
```

</details>

### Шаблон проектирования «Состояние типа» (Type State Pattern) и обобщения в Rust
- Типы Rust можно использовать для обеспечения правильности переходов конечного автомата на этапе *компиляции*.
    - Рассмотрим дрон (`Drone`) с двумя состояниями: `Idle` (ожидание) и `Flying` (полёт). В состоянии `Idle` единственным разрешенным методом является `takeoff()` (взлет). В состоянии `Flying` разрешен метод `land()` (посадка).
    
- Один из подходов — смоделировать конечный автомат следующим образом:
```rust
enum DroneState {
    Idle,
    Flying
}
struct Drone {x: u64, y: u64, z: u64, state: DroneState}  // x, y, z — это координаты
```
- Это требует множества проверок во время выполнения для соблюдения семантики конечного автомата — [▶ попробуйте сами](https://play.rust-lang.org/), чтобы понять почему.

### Обобщения в шаблоне «Состояние типа»
- Обобщения позволяют нам проверять логику конечного автомата во время *компиляции*. Для этого требуется использовать специальный маркерный тип `PhantomData<T>`.
- `PhantomData<T>` — это маркерный тип данных нулевого размера (zero-sized type). В данном случае мы используем его для представления состояний `Idle` и `Flying`, но он занимает `ноль` байт в памяти во время выполнения.
- Обратите внимание, что методы `takeoff` и `land` принимают `self` в качестве параметра. Это называется *потреблением* (в отличие от `&self`, который использует заимствование). Фактически, вызвав метод `takeoff()` для объекта `Drone<Idle>`, мы получаем взамен `Drone<Flying>`, а старый объект прекращает свое существование, и наоборот.
```rust
struct Drone<T> {x: u64, y: u64, z: u64, state: PhantomData<T> }
impl Drone<Idle> {
    fn takeoff(self) -> Drone<Flying> {...}
}
impl Drone<Flying> {
    fn land(self) -> Drone<Idle> { ...}
}
```
    - [▶ Попробовать в Rust Playground](https://play.rust-lang.org/)

### Обобщения в шаблоне «Состояние типа»
- Ключевые выводы:
    - Состояния могут быть представлены в виде структур нулевого размера (zero-size).
    - Мы можем комбинировать состояние `T` с `PhantomData<T>` (нулевого размера).
    - Реализация методов для определенной стадии конечного автомата теперь сводится к написанию `impl Drone<State>` (например, `impl Drone<Idle>`).
    - Используйте метод, который потребляет `self`, для перехода из одного состояния в другое.
    - Это дает нам абстракции с нулевой стоимостью (zero cost abstractions). Компилятор может контролировать конечный автомат во время компиляции, делая невозможным вызов методов, если объект находится в неверном состоянии.

### Шаблон проектирования «Строитель» (Builder Pattern) в Rust
- Потребление `self` может быть очень полезно для реализации шаблона «Строитель».
- Рассмотрим конфигурацию GPIO с несколькими десятками контактов (пинов). Контакты могут быть настроены в состояние высокого (high) или низкого (low) уровня (по умолчанию — низкий).
```rust
#[derive(default)]
enum PinState {
    #[default]
    Low,
    High,
} 
#[derive(default)]
struct GPIOConfig {
    pin0: PinState,
    pin1: PinState
    ... 
}
```
- Шаблон «Строитель» можно использовать для создания конфигурации GPIO с помощью цепочки вызовов методов — [▶ Попробовать](https://play.rust-lang.org/).
