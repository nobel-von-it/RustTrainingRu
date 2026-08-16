## Справочник по мощному инструментарию итераторов

> **Чему вы научитесь:** Продвинутые комбинаторы итераторов, выходящие за рамки `filter`/`map`/`collect` — `enumerate`, `zip`, `chain`, `flat_map`, `scan`, `windows` и `chunks`. Необходимы для замены индексных циклов `for` в стиле C безопасными и выразительными итераторами Rust.

Базовая цепочка `filter`/`map`/`collect` покрывает множество случаев, но библиотека итераторов Rust намного богаче. В этом разделе рассматриваются инструменты, к которым вы будете обращаться ежедневно — особенно при переносе циклов на C, которые вручную отслеживают индексы, накапливают результаты или обрабатывают данные фиксированными блоками.

### Быстрый справочник

| Метод | Эквивалент в C | Что делает | Возвращает |
|--------|-------------|-------------|---------|
| `enumerate()` | `for (int i=0; ...)` | Сопоставляет каждый элемент с его индексом | `(usize, T)` |
| `zip(other)` | Параллельные массивы с общим индексом | Объединяет элементы двух итераторов в пары | `(A, B)` |
| `chain(other)` | Обработка сначала array1, затем array2 | Последовательно соединяет два итератора | `T` |
| `flat_map(f)` | Вложенные циклы | Применяет отображение, затем уплотняет на один уровень | `U` |
| `windows(n)` | `for (int i=0; i<len-n+1; i++) &arr[i..i+n]` | Пересекающиеся срезы размера `n` | `&[T]` |
| `chunks(n)` | Обработка `n` элементов за раз | Непересекающиеся срезы размера `n` | `&[T]` |
| `fold(init, f)` | `int acc = init; for (...) acc = f(acc, x);` | Сведение к одному значению | `Acc` |
| `scan(init, f)` | Накопление с выводом промежуточных результатов | Как `fold`, но возвращает промежуточные результаты | `Option<B>` |
| `take(n)` / `skip(n)` | Начало цикла со смещением / ограничением | Первые `n` элементов / пропуск первых `n` | `T` |
| `take_while(f)` / `skip_while(f)` | `while (pred) {...}` | Выборка/пропуск, пока условие истинно | `T` |
| `peekable()` | Опережающий просмотр без извлечения | Позволяет вызывать `.peek()` без потребления | `T` |
| `step_by(n)` | `for (i=0; i<len; i+=n)` | Выборка каждого n-го элемента | `T` |
| `unzip()` | Разделение параллельных массивов | Разделение кортежей в две коллекции | `(A, B)` |
| `sum()` / `product()` | Вычисление суммы/произведения | Сведение с помощью `+` или `*` | `T` |
| `min()` / `max()` | Поиск экстремумов | Возвращает Option<T> | `Option<T>` |
| `any(f)` / `all(f)` | `bool found = false; for (...) ...` | Поиск с сокращенным вычислением (short-circuit) | `bool` |
| `position(f)` | `for (i=0; ...) if (pred) return i;` | Индекс первого совпадения | `Option<usize>` |

### `enumerate` — индекс + значение (заменяет индексные циклы C)

```rust
fn main() {
    let sensors = ["GPU_TEMP", "CPU_TEMP", "FAN_RPM", "PSU_WATT"];

    // Стиль C: for (int i = 0; i < 4; i++) printf("[%d] %s\n", i, sensors[i]);
    for (i, name) in sensors.iter().enumerate() {
        println!("[{i}] {name}");
    }

    // Найти индекс конкретного датчика
    let gpu_idx = sensors.iter().position(|&s| s == "GPU_TEMP");
    println!("GPU sensor at index: {gpu_idx:?}");  // Some(0)
}
```

### `zip` — параллельный обход (заменяет циклы параллельных массивов)

```rust
fn main() {
    let names = ["accel_diag", "nic_diag", "cpu_diag"];
    let statuses = [true, false, true];
    let durations_ms = [1200, 850, 3400];

    // Стиль C: for (int i=0; i<3; i++) printf("%s: %s (%d ms)\n", names[i], ...);
    for ((name, passed), ms) in names.iter().zip(&statuses).zip(&durations_ms) {
        let status = if *passed { "PASS" } else { "FAIL" };
        println!("{name}: {status} ({ms} ms)");
    }
}
```

### `chain` — последовательное соединение итераторов

```rust
fn main() {
    let critical = vec!["ECC error", "Thermal shutdown"];
    let warnings = vec!["Link degraded", "Fan slow"];

    // Обработка всех событий в порядке приоритета
    let all_events: Vec<_> = critical.iter().chain(warnings.iter()).collect();
    println!("{all_events:?}");
    // ["ECC error", "Thermal shutdown", "Link degraded", "Fan slow"]
}
```

### `flat_map` — уплотнение вложенных результатов

```rust
fn main() {
    let lines = vec!["gpu:42:ok", "nic:99:fail", "cpu:7:ok"];

    // Извлечение всех числовых значений из строк с разделителем `:`
    let numbers: Vec<u32> = lines.iter()
        .flat_map(|line| line.split(':'))
        .filter_map(|token| token.parse::<u32>().ok())
        .collect();
    println!("{numbers:?}");  // [42, 99, 7]
}
```

### `windows` и `chunks` — скользящие и фиксированные группы

```rust
fn main() {
    let temps = [65, 68, 72, 71, 75, 80, 78, 76];

    // windows(3): перекрывающиеся группы по 3 (например, для скользящего среднего)
    // Стиль C: for (int i = 0; i <= len-3; i++) avg(arr[i], arr[i+1], arr[i+2]);
    let moving_avg: Vec<f64> = temps.windows(3)
        .map(|w| w.iter().sum::<i32>() as f64 / 3.0)
        .collect();
    println!("Moving avg: {moving_avg:.1?}");

    // chunks(2): непересекающиеся группы по 2
    // Стиль C: for (int i = 0; i < len; i += 2) process(arr[i], arr[i+1]);
    for pair in temps.chunks(2) {
        println!("Chunk: {pair:?}");
    }

    // chunks_exact(2): то же самое, но вызывает панику при наличии остатка
    // Также: .remainder() возвращает оставшиеся элементы
}
```

### `fold` и `scan` — накопление

```rust
fn main() {
    let values = [10, 20, 30, 40, 50];

    // fold: один конечный результат (как цикл накопления в C)
    let sum = values.iter().fold(0, |acc, &x| acc + x);
    println!("Sum: {sum}");  // 150

    // Сборка строки с помощью fold
    let csv = values.iter()
        .fold(String::new(), |acc, x| {
            if acc.is_empty() { format!("{x}") }
            else { format!("{acc},{x}") }
        });
    println!("CSV: {csv}");  // "10,20,30,40,50"

    // scan: как fold, но возвращает промежуточные результаты
    let running_sum: Vec<i32> = values.iter()
        .scan(0, |state, &x| {
            *state += x;
            Some(*state)
        })
        .collect();
    println!("Running sum: {running_sum:?}");  // [10, 30, 60, 100, 150]
}
```

### Упражнение: Конвейер данных с датчиков

Даны необработанные показания датчиков (по одному в строке, формат `"sensor_name:value:unit"`). Напишите конвейер итераторов, который:
1. Парсит каждую строку в `(name, f64, unit)`
2. Отфильтровывает показания ниже порогового значения
3. Группирует по имени датчика с помощью `fold` в `HashMap`
4. Выводит среднее значение показаний для каждого датчика

```rust
// Начальный код
fn main() {
    let raw_data = vec![
        "gpu_temp:72.5:C",
        "cpu_temp:65.0:C",
        "gpu_temp:74.2:C",
        "fan_rpm:1200.0:RPM",
        "cpu_temp:63.8:C",
        "gpu_temp:80.1:C",
        "fan_rpm:1150.0:RPM",
    ];
    let threshold = 70.0;
    // TODO: Парсинг, фильтрация значений >= threshold, группировка по имени, вычисление средних значений
}
```

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
use std::collections::HashMap;

fn main() {
    let raw_data = vec![
        "gpu_temp:72.5:C",
        "cpu_temp:65.0:C",
        "gpu_temp:74.2:C",
        "fan_rpm:1200.0:RPM",
        "cpu_temp:63.8:C",
        "gpu_temp:80.1:C",
        "fan_rpm:1150.0:RPM",
    ];
    let threshold = 70.0;

    // Парсинг → фильтрация → группировка → усреднение
    let grouped = raw_data.iter()
        .filter_map(|line| {
            let parts: Vec<&str> = line.splitn(3, ':').collect();
            if parts.len() == 3 {
                let value: f64 = parts[1].parse().ok()?;
                Some((parts[0], value, parts[2]))
            } else {
                None
            }
        })
        .filter(|(_, value, _)| *value >= threshold)
        .fold(HashMap::<&str, Vec<f64>>::new(), |mut acc, (name, value, _)| {
            acc.entry(name).or_default().push(value);
            acc
        });

    for (name, values) in &grouped {
        let avg = values.iter().sum::<f64>() / values.len() as f64;
        println!("{name}: avg={avg:.1} ({} readings)", values.len());
    }
}
// Вывод (порядок может отличаться):
// gpu_temp: avg=75.6 (3 readings)
// fan_rpm: avg=1175.0 (2 readings)
```

</details>


# Итераторы в Rust
- Трейт `Iterator` используется для реализации итерации по пользовательским типам (https://doc.rust-lang.org/std/iter/trait.IntoIterator.html).
    - В этом примере мы реализуем итератор для последовательности Фибоначчи, которая начинается с 1, 1, 2, ... и каждое последующее число равно сумме двух предыдущих.
    - Ассоциированный тип в трейте `Iterator` (`type Item = u32;`) определяет тип элементов, выдаваемых итератором (`u32`).
    - Метод `next()` содержит логику работы итератора. В данном случае вся информация о состоянии хранится в структуре `Fibonacci`.
    - Мы также могли бы реализовать трейт `IntoIterator`, чтобы предоставить метод `into_iter()` для более специализированных итераторов.
    - https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=ab367dc2611e1b5a0bf98f1185b38f3f
