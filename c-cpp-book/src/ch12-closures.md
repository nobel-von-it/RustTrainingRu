## Замыкания в Rust (Closures)

> **Чему вы научитесь:** Замыкания как анонимные функции, три трейта захвата переменных (`Fn`, `FnMut`, `FnOnce`), замыкания с перемещением (`move`), а также сравнение замыканий Rust с лямбда-выражениями C++ (автоматический анализ захвата вместо явного указания `[&]` / `[=]`).

- Замыкания — это анонимные функции, которые могут захватывать переменные из своего окружения.
    - Эквивалент в C++: лямбда-выражения (`[&](int x) { return x + 1; }`).
    - Ключевое отличие: замыкания в Rust имеют **три** трейта захвата (`Fn`, `FnMut`, `FnOnce`), которые компилятор выбирает автоматически.
    - Режимы захвата в C++ (`[=]`, `[&]`, `[this]`) задаются вручную и подвержены ошибкам (например, висячие ссылки при захвате по ссылке `[&]`).
    - Проверка заимствований (borrow checker) в Rust предотвращает появление висячих ссылок в захваченных переменных еще на этапе компиляции.
- Замыкания обозначаются символом `||`. Параметры и их типы заключаются внутри `||`, при этом типы параметров могут выводиться автоматически.
- Замыкания часто используются в сочетании с итераторами (следующая тема).
```rust
fn add_one(x: u32) -> u32 {
    x + 1
}
fn main() {
    let add_one_v1 = |x : u32| {x + 1}; // Явно указанный тип
    let add_one_v2 = |x| {x + 1};   // Тип выводится из контекста вызова
    let add_one_v3 = |x| x+1;   // Разрешено для однострочных функций
    println!("{} {} {} {}", add_one(42), add_one_v1(42), add_one_v2(42), add_one_v3(42) );
}
```


# Упражнение: Замыкания и захват переменных

🟡 **Средний уровень**

- Создайте замыкание, которое захватывает строку `String` из внешней области видимости и добавляет к ней текст (подсказка: используйте `move`).
- Создайте вектор замыканий `Vec<Box<dyn Fn(i32) -> i32>>`, содержащий замыкания, которые прибавляют 1, умножают на 2 и возводят входное число в квадрат. Переберите вектор в цикле и примените каждое замыкание к числу 5.

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
fn main() {
    // Часть 1: Замыкание, которое захватывает и дополняет String
    let mut greeting = String::from("Hello");
    let mut append = |suffix: &str| {
        greeting.push_str(suffix);
    };
    append(", world");
    append("!");
    println!("{greeting}");  // "Hello, world!"

    // Часть 2: Вектор замыканий
    let operations: Vec<Box<dyn Fn(i32) -> i32>> = vec![
        Box::new(|x| x + 1),      // прибавить 1
        Box::new(|x| x * 2),      // умножить на 2
        Box::new(|x| x * x),      // возвести в квадрат
    ];

    let input = 5;
    for (i, op) in operations.iter().enumerate() {
        println!("Operation {i} on {input}: {}", op(input));
    }
}
// Вывод:
// Hello, world!
// Operation 0 on 5: 6
// Operation 1 on 5: 10
// Operation 2 on 5: 25
```

</details>

# Итераторы в Rust
- Итераторы — одна из самых мощных возможностей Rust. Они позволяют элегантно выполнять операции над коллекциями, включая фильтрацию (`filter()`), преобразование (`map()`), совмещенную фильтрацию и преобразование (`filter_map()`), поиск (`find()`) и многое другое.
- В приведенном ниже примере `|&x| *x >= 42` — это замыкание, выполняющее сравнение. `|x| println!("{x}")` — еще одно замыкание.
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    for x in &a {
        if *x >= 42 {
            println!("{x}");
        }
    }
    // То же самое, что и выше
    a.iter().filter(|&x| *x >= 42).for_each(|x| println!("{x}"))
}
```

# Итераторы в Rust
- Важной особенностью итераторов является их ленивость (lazy): они ничего не делают, пока их не вычислят (не вызовут терминальный метод). Например, вызов `a.iter().filter(|&x| *x >= 42);` не выполнил бы *никакой* работы без метода вроде `for_each`. Компилятор Rust выдает предупреждение, когда обнаруживает подобную ситуацию.
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    // Добавить единицу к каждому элементу и вывести его
    let _ = a.iter().map(|x|x + 1).for_each(|x|println!("{x}"));
    let found = a.iter().find(|&x|*x == 42);
    println!("{found:?}");
    // Подсчитать элементы
    let count = a.iter().count();
    println!("{count}");
}
```

# Итераторы в Rust
- Метод `collect()` используется для сбора результатов работы итератора в отдельную коллекцию.
    - Символ `_` в выражении `Vec<_>` ниже играет роль шаблона подстановки для типа, возвращаемого методом `map`. Например, мы можем возвращать из `map` даже тип `String`.
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    let squared_a : Vec<_> = a.iter().map(|x|x*x).collect();
    for x in &squared_a {
        println!("{x}");
    }
    let squared_a_strings : Vec<_> = a.iter().map(|x|(x*x).to_string()).collect();
    // Это строковые представления чисел
    for x in &squared_a_strings {
        println!("{x}");
    }
}
```

# Упражнение: Итераторы в Rust

🟢 **Начальный уровень**
- Создайте массив целых чисел, состоящий из четных и нечетных элементов. Переберите его элементами и разделите на два разных вектора: один с четными, другой с нечетными элементами.
- Можно ли сделать это за один проход (подсказка: используйте `partition()`)?

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Подход 1: Ручной перебор
    let mut evens = Vec::new();
    let mut odds = Vec::new();
    for n in numbers {
        if n % 2 == 0 {
            evens.push(n);
        } else {
            odds.push(n);
        }
    }
    println!("Evens: {evens:?}");
    println!("Odds:  {odds:?}");

    // Подход 2: Один проход с использованием partition()
    let (evens, odds): (Vec<i32>, Vec<i32>) = numbers
        .into_iter()
        .partition(|n| *n % 2 == 0);
    println!("Evens (partition): {evens:?}");
    println!("Odds  (partition): {odds:?}");
}
// Вывод:
// Evens: [2, 4, 6, 8, 10]
// Odds:  [1, 3, 5, 7, 9]
// Evens (partition): [2, 4, 6, 8, 10]
// Odds  (partition): [1, 3, 5, 7, 9]
```

</details>

> **Примеры из реального кода**: См. раздел [Упрощение пирамид присваивания с помощью замыканий](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids-with-closures) для ознакомления с реальными цепочками итераторов (`.map().collect()`, `.filter().collect()`, `.find_map()`) из промышленного кода на Rust.

### Мощный инструментарий итераторов: методы, заменяющие циклы C++

Следующие адаптеры итераторов *очень часто* используются в промышленном коде на Rust. В C++ есть заголовочный файл `<algorithm>` и концепция Ranges из C++20, но цепочки итераторов в Rust лучше поддаются композиции и используются гораздо шире.

#### `enumerate` — индекс + значение (заменяет `for (int i = 0; ...)`)

```rust
let sensors = vec!["temp0", "temp1", "temp2"];
for (idx, name) in sensors.iter().enumerate() {
    println!("Sensor {idx}: {name}");
}
// Sensor 0: temp0
// Sensor 1: temp1
// Sensor 2: temp2
```

Эквивалент в C++: `for (size_t i = 0; i < sensors.size(); ++i) { auto& name = sensors[i]; ... }`

#### `zip` — объединение элементов двух итераторов в пары (заменяет циклы с параллельными индексами)

```rust
let names = ["gpu0", "gpu1", "gpu2"];
let temps = [72.5, 68.0, 75.3];

let report: Vec<String> = names.iter()
    .zip(temps.iter())
    .map(|(name, temp)| format!("{name}: {temp}°C"))
    .collect();
println!("{report:?}");
// ["gpu0: 72.5°C", "gpu1: 68.0°C", "gpu2: 75.3°C"]

// Итерация останавливается по более короткому итератору — нет риска выхода за границы диапазона
```

Эквивалент в C++: `for (size_t i = 0; i < std::min(names.size(), temps.size()); ++i) { ... }`

#### `flat_map` — преобразование (map) + уплотнение (flatten) вложенных коллекций

```rust
// Каждый графический процессор (GPU) имеет несколько PCIe BDF; соберем все BDF для всех GPU
let gpu_bdfs = vec![
    vec!["0000:01:00.0", "0000:02:00.0"],
    vec!["0000:41:00.0"],
    vec!["0000:81:00.0", "0000:82:00.0"],
];

let all_bdfs: Vec<&str> = gpu_bdfs.iter()
    .flat_map(|bdfs| bdfs.iter().copied())
    .collect();
println!("{all_bdfs:?}");
// ["0000:01:00.0", "0000:02:00.0", "0000:41:00.0", "0000:81:00.0", "0000:82:00.0"]
```

Эквивалент в C++: вложенный цикл `for`, добавляющий элементы в один вектор.

#### `chain` — последовательное соединение двух итераторов

```rust
let critical_gpus = vec!["gpu0", "gpu3"];
let warning_gpus = vec!["gpu1", "gpu5"];

// Обработка всех отмеченных GPU, критические — в первую очередь
for gpu in critical_gpus.iter().chain(warning_gpus.iter()) {
    println!("Flagged: {gpu}");
}
```

#### `windows` и `chunks` — скользящие и фиксированные окна для срезов

```rust
let temps = [70, 72, 75, 73, 71, 68, 65];

// windows(3): скользящее окно размера 3 — обнаружение тенденций
let rising = temps.windows(3)
    .any(|w| w[0] < w[1] && w[1] < w[2]);
println!("Rising trend detected: {rising}"); // true (70 < 72 < 75)

// chunks(2): группы фиксированного размера — обработка парами
for pair in temps.chunks(2) {
    println!("Pair: {pair:?}");
}
// Pair: [70, 72]
// Pair: [75, 73]
// Pair: [71, 68]
// Pair: [65]       ← последний элемент (чанк) может быть меньше
```

Эквивалент в C++: ручная индексация и арифметические операции с `i` и `i+1`/`i+2`.

#### `fold` — сведение в единое значение (заменяет `std::accumulate`)

```rust
let errors = vec![
    ("gpu0", 3u32),
    ("gpu1", 0),
    ("gpu2", 7),
    ("gpu3", 1),
];

// Подсчитаем общее число ошибок и соберем сводку за один проход
let (total, summary) = errors.iter().fold(
    (0u32, String::new()),
    |(count, mut s), (name, errs)| {
        if *errs > 0 {
            s.push_str(&format!("{name}:{errs} "));
        }
        (count + errs, s)
    },
);
println!("Total errors: {total}, details: {summary}");
// Total errors: 11, details: gpu0:3 gpu2:7 gpu3:1
```

#### `scan` — преобразование с сохранением состояния (промежуточные суммы, выявление разницы)

```rust
let readings = [100, 105, 103, 110, 108];

// Вычисление разницы между последовательными показаниями
let deltas: Vec<i32> = readings.iter()
    .scan(None::<i32>, |prev, &val| {
        let delta = prev.map(|p| val - p);
        *prev = Some(val);
        Some(delta)
    })
    .flatten()  // Удаление начального None
    .collect();
println!("Deltas: {deltas:?}"); // [5, -2, 7, -2]
```

#### Быстрый справочник: Перевод циклов C++ в итераторы Rust

| **Шаблон C++** | **Итератор Rust** | **Пример** |
|----------------|------------------|------------|
| `for (int i = 0; i < v.size(); i++)` | `.enumerate()` | `v.iter().enumerate()` |
| Параллельный обход с индексом | `.zip()` | `a.iter().zip(b.iter())` |
| Вложенный цикл → плоский результат | `.flat_map()` | `vecs.iter().flat_map(\|v\| v.iter())` |
| Объединение двух контейнеров | `.chain()` | `a.iter().chain(b.iter())` |
| Скользящее окно `v[i..i+n]` | `.windows(n)` | `v.windows(3)` |
| Обработка группами фиксированного размера | `.chunks(n)` | `v.chunks(4)` |
| `std::accumulate` / ручное накопление | `.fold()` | `.fold(init, \|acc, x\| ...)` |
| Промежуточный итог / отслеживание разницы | `.scan()` | `.scan(state, \|s, x\| ...)` |
| `while (it != end && count < n) { ++it; ++count; }` | `.take(n)` | `.iter().take(5)` |
| `while (it != end && !pred(*it)) { ++it; }` | `.skip_while()` | `.skip_while(\|x\| x < &threshold)` |
| `std::any_of` | `.any()` | `.iter().any(\|x\| x > &limit)` |
| `std::all_of` | `.all()` | `.iter().all(\|x\| x.is_valid())` |
| `std::none_of` | `!.any()` | `!iter.any(\|x\| x.failed())` |
| `std::count_if` | `.filter().count()` | `.filter(\|x\| x > &0).count()` |
| `std::min_element` / `std::max_element` | `.min()` / `.max()` | `.iter().max()` → `Option<&T>` |
| `std::unique` | `.dedup()` (на отсортированном) | `v.dedup()` (in-place для Vec) |

### Упражнение: Цепочки итераторов

Даны показания датчиков в виде `Vec<(String, f64)>` (имя, температура). Напишите **одну цепочку итераторов**, которая:
1. Фильтрует датчики с температурой > 80.0
2. Сортирует их по температуре (по убыванию)
3. Форматирует каждый элемент как `"{name}: {temp}°C [ALARM]"`
4. Собирает результат в `Vec<String>`

Подсказка: вам потребуется вызвать `.collect()` перед `.sort_by()`, поскольку для сортировки необходим `Vec`.

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
fn alarm_report(sensors: &[(String, f64)]) -> Vec<String> {
    let mut hot: Vec<_> = sensors.iter()
        .filter(|(_, temp)| *temp > 80.0)
        .collect();
    hot.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    hot.iter()
        .map(|(name, temp)| format!("{name}: {temp}°C [ALARM]"))
        .collect()
}

fn main() {
    let sensors = vec![
        ("gpu0".to_string(), 72.5),
        ("gpu1".to_string(), 85.3),
        ("gpu2".to_string(), 91.0),
        ("gpu3".to_string(), 78.0),
        ("gpu4".to_string(), 88.7),
    ];
    for line in alarm_report(&sensors) {
        println!("{line}");
    }
}
// Вывод:
// gpu2: 91°C [ALARM]
// gpu4: 88.7°C [ALARM]
// gpu1: 85.3°C [ALARM]
```

</details>

----

# Итераторы в Rust
- Трейт `Iterator` используется для реализации итерации по пользовательским типам (https://doc.rust-lang.org/std/iter/trait.IntoIterator.html).
    - В этом примере мы реализуем итератор для последовательности Фибоначчи, которая начинается с 1, 1, 2, ... и каждое последующее число равно сумме двух предыдущих.
    - Ассоциированный тип в трейте `Iterator` (`type Item = u32;`) определяет тип элементов, выдаваемых итератором (`u32`).
    - Метод `next()` содержит логику работы итератора. В данном случае вся информация о состоянии хранится в структуре `Fibonacci`.
    - Мы также могли бы реализовать трейт `IntoIterator`, чтобы предоставить метод `into_iter()` для более специализированных итераторов.
    - [▶ Попробовать в Rust Playground](https://play.rust-lang.org/)
