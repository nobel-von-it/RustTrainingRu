# Трейты From и Into в Rust

> **Чему вы научитесь:** Трейты преобразования типов в Rust — `From<T>` и `Into<T>` для безошибочных преобразований, `TryFrom` и `TryInto` для потенциально аварийных. Реализуйте `From` и получите `Into` автоматически. Заменяет операторы преобразования и конструкторы C++.

- `From` и `Into` — это взаимодополняющие трейты, облегчающие преобразование типов.
- Обычно типы реализуют трейт `From`. Например, `String::from("Rust")` преобразует `&str` в `String`.
Если существует реализация `From<T> for U`, Rust автоматически предоставляет реализацию `Into<U> for T`, поэтому код `let s: String = "Rust".into();` также работает.
```rust
struct Point {x: u32, y: u32}
// Создание точки Point из кортежа
impl From<(u32, u32)> for Point {
    fn from(xy : (u32, u32)) -> Self {
        Point {x : xy.0, y: xy.1}       // Создаем Point, используя элементы кортежа
    }
}
fn main() {
    let s = String::from("Rust");
    let x = u32::from(true);
    let p = Point::from((40, 42));
    // let p : Point = (40,42).into(); // Альтернативная форма для строки выше
    println!("s: {s} x:{x} p.x:{} p.y {}", p.x, p.y);   
}
```

# Упражнение: From и Into
- Реализуйте трейт `From` для структуры `Point`, чтобы преобразовывать ее в тип `TransposePoint`. `TransposePoint` меняет местами поля `x` и `y` структуры `Point`.

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
struct Point { x: u32, y: u32 }
struct TransposePoint { x: u32, y: u32 }

impl From<Point> for TransposePoint {
    fn from(p: Point) -> Self {
        TransposePoint { x: p.y, y: p.x }
    }
}

fn main() {
    let p = Point { x: 10, y: 20 };
    let tp = TransposePoint::from(p);
    println!("Transposed: x={}, y={}", tp.x, tp.y);  // x=20, y=10

    // Использование .into() — работает автоматически, когда реализован From
    let p2 = Point { x: 3, y: 7 };
    let tp2: TransposePoint = p2.into();
    println!("Transposed: x={}, y={}", tp2.x, tp2.y);  // x=7, y=3
}
// Вывод:
// Transposed: x=20, y=10
// Transposed: x=7, y=3
```

</details>

# Трейт Default в Rust
- Трейт `Default` используется для задания значений по умолчанию для определенного типа.
    - Типы могут использовать макрос автовывода `Derive` с `Default` или предоставлять собственную реализацию вручную.
```rust
#[derive(Default, Debug)]
struct Point {x: u32, y: u32}
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = Point::default();   // Создает Point{0, 0}
    println!("{x:?}");
    let y = CustomPoint::default();
    println!("{y:?}");
}
```

### Трейт Default в Rust
- Трейт `Default` имеет несколько вариантов применения, включая:
    - Выполнение частичного копирования и использование инициализации по умолчанию для остальных полей.
    - Альтернативное значение по умолчанию для типов `Option` в методах вроде `unwrap_or_default()`.
```rust
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = CustomPoint::default();
    // Переопределяем y, но оставляем остальные элементы по умолчанию
    let y = CustomPoint {y: 43, ..CustomPoint::default()};
    println!("{x:?} {y:?}");
    let z : Option<CustomPoint> = None;
    // Попробуйте изменить unwrap_or_default() на unwrap()
    println!("{:?}", z.unwrap_or_default());
}
```

### Другие преобразования типов в Rust
- Rust не поддерживает неявные преобразования типов, а оператор `as` может использоваться для *явных* преобразований.
- Оператор `as` следует использовать с осторожностью, поскольку он подвержен риску потери данных при сужении типов и т. п. В целом, по возможности предпочтительнее использовать методы `into()` или `from()` (если они доступны).
```rust
fn main() {
    let f = 42u8;
    // let g : u32 = f;    // Не скомпилируется
    let g = f as u32;      // OK, но не предпочтительно. Подпадает под правила сужения типов
    let g : u32 = f.into(); // Наиболее предпочтительная форма: безопасная (без паники) и проверяемая компилятором
    // let k : u8 = g.into();  // Ошибка компиляции: сужение типа может привести к потере данных
    
    // Попытка сужения типа требует использования try_into
    if let Ok(k) = TryInto::<u8>::try_into(g) {
        println!("{k}");
    }
}
```
