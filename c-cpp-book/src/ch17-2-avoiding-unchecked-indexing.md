## Как избегать небезопасного индексирования

> **Чему вы научитесь:** Почему обращение `vec[i]` опасно в Rust (вызывает панику при выходе за границы), и о безопасных альтернативах, таких как `.get()`, итераторы и API `entry()` для `HashMap`. Замена неопределенного поведения C++ на явную обработку.

- В C++ обращения `vec[i]` и `map[key]` приводят к неопределенному поведению или автоматической вставке при отсутствии ключей. В Rust оператор `[]` вызывает панику при выходе за границы диапазона.
- **Правило**: используйте метод `.get()` вместо `[]`, если только вы не можете точно *доказать*, что индекс валиден.

### Сравнение C++ и Rust
```cpp
// C++ — тихое неопределенное поведение (UB) или вставка
std::vector<int> v = {1, 2, 3};
int x = v[10];        // UB! Отсутствует проверка границ в operator[]

std::map<std::string, int> m;
int y = m["missing"]; // Тихо вставляет ключ со значением 0!
```

```rust
// Rust — безопасные альтернативы
let v = vec![1, 2, 3];

// Плохо: паникует, если индекс вне границ
// let x = v[10];

// Хорошо: возвращает Option<&i32>
let x = v.get(10);              // None — без паники
let x = v.get(1).copied().unwrap_or(0);  // 2, или 0 в случае отсутствия
```

### Реальный пример: безопасный парсинг байтов из продакшн-кода Rust
```rust
// Пример: diagnostics.rs
// Парсинг бинарной записи SEL — буфер может оказаться короче, чем ожидается
let sensor_num = bytes.get(7).copied().unwrap_or(0);
let ppin = cpu_ppin.get(i).map(|s| s.as_str()).unwrap_or("");
```

### Реальный пример: цепочка безопасных поисков с `.and_then()`
```rust
// Пример: profile.rs — двойной поиск: HashMap → Vec
pub fn get_processor(&self, location: &str) -> Option<&Processor> {
    self.processor_by_location
        .get(location)                              // HashMap → Option<&usize>
        .and_then(|&idx| self.processors.get(idx))   // Vec → Option<&Processor>
}
// Оба поиска возвращают Option — никаких паник или UB
```

### Реальный пример: безопасная навигация по JSON
```rust
// Пример: framework.rs — каждый ключ JSON возвращает Option
let manufacturer = product_fru
    .get("Manufacturer")            // Option<&Value>
    .and_then(|v| v.as_str())       // Option<&str>
    .unwrap_or(UNKNOWN_VALUE)       // &str (безопасный резервный вариант)
    .to_string();
```
Сравните с шаблоном в C++: `json["SystemInfo"]["ProductFru"]["Manufacturer"]` — любой отсутствующий ключ сгенерирует исключение `nlohmann::json::out_of_range`.

### Когда использование `[]` допустимо
- **После проверки границ**: `if i < v.len() { v[i] }`
- **В тестах**: где возникновение паники является ожидаемым поведением.
- **С константами**: `let first = v[0];` сразу после `assert!(!v.is_empty());`.

----

## Безопасное извлечение значений с помощью unwrap_or

- Метод `unwrap()` паникует в случае `None` / `Err`. В продакшн-коде отдавайте предпочтение безопасным альтернативам.

### Семейство методов unwrap
| **Метод** | **Поведение при None/Err** | **Когда использовать** |
|-----------|------------------------|-------------|
| `.unwrap()` | **Паникует** | Только в тестах или при гарантированной невозможности ошибки |
| `.expect("msg")` | Паникует с сообщением | Когда паника оправдана, объясните почему |
| `.unwrap_or(default)` | Возвращает `default` | У вас есть простое константное значение по умолчанию |
| `.unwrap_or_else(\|\| expr)` | Вызывает замыкание | Вычисление значения по умолчанию ресурсоемко |
| `.unwrap_or_default()` | Возвращает `Default::default()` | Тип реализует типаж `Default` |

### Реальный пример: парсинг с безопасными значениями по умолчанию
```rust
// Пример: peripherals.rs
// Группы захвата регулярного выражения могут не совпасть — предоставляем безопасные резервные значения
let bus_hex = caps.get(1).map(|m| m.as_str()).unwrap_or("00");
let fw_status = caps.get(5).map(|m| m.as_str()).unwrap_or("0x0");
let bus = u8::from_str_radix(bus_hex, 16).unwrap_or(0);
```

### Реальный пример: `unwrap_or_else` со структурой по умолчанию
```rust
// Пример: framework.rs
// Основная функция оборачивает логику в замыкание, возвращающее Option;
// если происходит сбой, возвращаем дефолтную структуру:
(|| -> Option<BaseboardFru> {
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    // ... извлекаем поля с помощью цепочек .get()?
    Some(baseboard_fru)
})()
.unwrap_or_else(|| BaseboardFru {
    manufacturer: String::new(),
    model: String::new(),
    product_part_number: String::new(),
    serial_number: String::new(),
    asset_tag: String::new(),
})
```

### Реальный пример: `unwrap_or_default` при десериализации конфигурации
```rust
// Пример: framework.rs
// Если парсинг конфигурации JSON завершается ошибкой, откатываемся к Default — без падения программы
Ok(json) => serde_json::from_str(&json).unwrap_or_default(),
```
Эквивалентом в C++ была бы конструкция `try/catch` вокруг вызова `nlohmann::json::parse()` с ручным конструированием объекта по умолчанию в блоке catch.

----

## Функциональные преобразования: map, map_err, find_map

- Эти методы типажей `Option` и `Result` позволяют преобразовывать содержащееся внутри значение без выполнения unwrap, заменяя вложенные конструкции `if/else` линейными цепочками.

### Сводная таблица
| **Метод** | **Для типа** | **Что делает** | **Эквивалент в C++** |
|-----------|-------|---------|-------------------|
| `.map(\|v\| ...)` | `Option` / `Result` | Преобразует значение `Some`/`Ok` | `if (opt) { *opt = transform(*opt); }` |
| `.map_err(\|e\| ...)` | `Result` | Преобразует значение `Err` | Добавление контекста в блок catch |
| `.and_then(\|v\| ...)` | `Option` / `Result` | Связывает операции, возвращающие `Option`/`Result` | Вложенные проверки if |
| `.find_map(\|v\| ...)` | Итератор | `find` + `map` за один проход | Цикл с проверкой `if + break` |
| `.filter(\|v\| ...)` | `Option` / Итератор | Сохраняет только значения, подходящие под предикат | `if (!predicate) return nullopt;` |
| `.ok()?` | `Result` | Преобразует `Result → Option` и пробрасывает `None` | `if (result.has_error()) return nullopt;` |

### Реальный пример: цепочка `.and_then()` для извлечения полей из JSON
```rust
// Пример: framework.rs — поиск серийного номера с запасными вариантами
let sys_info = json.get("SystemInfo")?;

// Сначала пробуем BaseboardFru.BoardSerialNumber
if let Some(serial) = sys_info
    .get("BaseboardFru")
    .and_then(|b| b.get("BoardSerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)     // Принимаем только непустые валидные серийные номера
{
    return Some(serial.to_string());
}

// Резервный вариант: BoardFru.SerialNumber
sys_info
    .get("BoardFru")
    .and_then(|b| b.get("SerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)
    .map(|s| s.to_string())   // Конвертируем &str → String только в случае Some
```
В C++ это выглядело бы как пирамида вида `if (json.contains("BaseboardFru")) { if (json["BaseboardFru"].contains("BoardSerialNumber")) { ... } }`.

### Реальный пример: `find_map` — поиск и преобразование за один проход
```rust
// Пример: context.rs — поиск записи SDR, соответствующей сенсору и владельцу
pub fn find_for_event(&self, sensor_number: u8, owner_id: u8) -> Option<&SdrRecord> {
    self.by_sensor.get(&sensor_number).and_then(|indices| {
        indices.iter().find_map(|&i| {
            let record = &self.records[i];
            if record.sensor_owner_id() == Some(owner_id) {
                Some(record)
            } else {
                None
            }
        })
    })
}
```
`find_map` представляет собой объединение функций `find` и `map`: метод останавливается на первом совпадении и преобразует его. Эквивалентом в C++ является цикл `for` с конструкцией `if` + `break`.

### Реальный пример: `map_err` для добавления контекста ошибки
```rust
// Пример: main.rs — добавление контекста к ошибкам перед пробросом
let json_str = serde_json::to_string_pretty(&config)
    .map_err(|e| format!("Не удалось сериализовать конфигурацию: {}", e))?;
```
Преобразует `serde_json::Error` в подробную текстовую ошибку типа `String`, описывающую, *что* именно пошло не так.

----

## Работа с JSON: перевод с nlohmann::json на serde

- Команды C++ обычно используют `nlohmann::json` для парсинга JSON. В Rust применяется связка **serde** + **serde_json**, которая гораздо эффективнее благодаря тому, что схема JSON кодируется непосредственно *в системе типов*.

### Сравнение: C++ (nlohmann) против Rust (serde)

```cpp
// C++ с nlohmann::json — доступ к полям во время выполнения
#include <nlohmann/json.hpp>
using json = nlohmann::json;

struct Fan {
    std::string logical_id;
    std::vector<std::string> sensor_ids;
};

Fan parse_fan(const json& j) {
    Fan f;
    f.logical_id = j.at("LogicalID").get<std::string>();    // выбрасывает исключение, если отсутствует
    if (j.contains("SDRSensorIdHexes")) {                   // ручная обработка значения по умолчанию
        f.sensor_ids = j["SDRSensorIdHexes"].get<std::vector<std::string>>();
    }
    return f;
}
```

```rust
// Rust с serde — схема на этапе компиляции, автоматическое сопоставление полей
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Fan {
    pub logical_id: String,
    #[serde(rename = "SDRSensorIdHexes", default)]  // Ключ JSON → поле Rust
    pub sensor_ids: Vec<String>,                     // Отсутствует → пустой Vec
    #[serde(default)]
    pub sensor_names: Vec<String>,                   // Отсутствует → пустой Vec
}

// Одна строка заменяет всю функцию парсинга:
let fan: Fan = serde_json::from_str(json_str)?;
```

### Ключевые атрибуты serde (реальные примеры из продакшн-кода Rust)

| **Атрибут** | **Назначение** | **Эквивалент в C++** |
|--------------|------------|--------------------|
| `#[serde(default)]` | Использует `Default::default()` при отсутствии полей | `if (j.contains(key)) { ... } else { default; }` |
| `#[serde(rename = "Key")]` | Сопоставляет имя ключа JSON с именем поля в Rust | Ручной доступ через `j.at("Key")` |
| `#[serde(flatten)]` | Собирает неизвестные ключи в `HashMap` | `for (auto& [k,v] : j.items()) { ... }` |
| `#[serde(skip)]` | Игнорирует поле при сериализации/десериализации | Не сохраняется в JSON |
| `#[serde(tag = "type")]` | Перечисление с внутренним тегом (поле-дискриминатор) | `if (j["type"] == "gpu") { ... }` |

### Реальный пример: полная структура конфигурации
```rust
// Пример: diag.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DiagConfig {
    pub sku: SkuConfig,
    #[serde(default)]
    pub level: DiagLevel,            // Отсутствует → DiagLevel::default()
    #[serde(default)]
    pub modules: ModuleConfig,       // Отсутствует → ModuleConfig::default()
    #[serde(default)]
    pub output_dir: String,          // Отсутствует → ""
    #[serde(default, flatten)]
    pub options: HashMap<String, serde_json::Value>,  // Поглощает неизвестные ключи
}

// Загрузка занимает 3 строки (против более чем 20 в C++ с использованием nlohmann):
let content = std::fs::read_to_string(path)?;
let config: DiagConfig = serde_json::from_str(&content)?;
Ok(config)
```

### Десериализация перечисления с `#[serde(tag = "type")]`
```rust
// Пример: components.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]                   // JSON: {"type": "Gpu", "product": ...}
pub enum PcieDeviceKind {
    Gpu { product: GpuProduct, manufacturer: GpuManufacturer },
    Nic { product: NicProduct, manufacturer: NicManufacturer },
    NvmeDrive { drive_type: StorageDriveType, capacity_gb: u32 },
    // ... еще 9 вариантов
}
// serde автоматически перенаправляет разбор на основе поля "type" — цепочка if/else вручную не нужна
```
Эквивалент в C++: `if (j["type"] == "Gpu") { parse_gpu(j); } else if (j["type"] == "Nic") { parse_nic(j); } ...`

# Упражнение: Десериализация JSON с помощью serde

- Определите структуру `ServerConfig`, которая может быть десериализована из следующего JSON:
```json
{
    "hostname": "diag-node-01",
    "port": 8080,
    "debug": true,
    "modules": ["accel_diag", "nic_diag", "cpu_diag"]
}
```
- Используйте `#[derive(Deserialize)]` и `serde_json::from_str()` для ее парсинга.
- Добавьте `#[serde(default)]` для поля `debug`, чтобы оно принимало значение по умолчанию `false`, если отсутствует в JSON.
- **Бонус**: добавьте поле с типом `enum DiagLevel { Quick, Full, Extended }` и атрибутом `#[serde(default)]`, чтобы оно по умолчанию инициализировалось значением `Quick`.

**Начальный код** (требует добавления зависимостей с помощью `cargo add serde --features derive` и `cargo add serde_json`):
```rust
use serde::Deserialize;

// TODO: Определить перечисление DiagLevel с реализацией Default

// TODO: Определить структуру ServerConfig с атрибутами serde

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    // TODO: Десериализовать и вывести конфигурацию
    // TODO: Попробовать распарсить JSON без поля "debug" — убедиться, что оно по умолчанию равно false
}
```

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize, Default)]
enum DiagLevel {
    #[default]
    Quick,
    Full,
    Extended,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    hostname: String,
    port: u16,
    #[serde(default)]       // по умолчанию равно false, если отсутствует
    debug: bool,
    modules: Vec<String>,
    #[serde(default)]       // по умолчанию равно DiagLevel::Quick, если отсутствует
    level: DiagLevel,
}

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    let config: ServerConfig = serde_json::from_str(json_input)
        .expect("Failed to parse JSON");
    println!("{config:#?}");

    // Тестируем с отсутствующими необязательными полями
    let minimal = r#"{
        "hostname": "node-02",
        "port": 9090,
        "modules": []
    }"#;
    let config2: ServerConfig = serde_json::from_str(minimal)
        .expect("Failed to parse minimal JSON");
    println!("debug (default): {}", config2.debug);    // false
    println!("level (default): {:?}", config2.level);  // Quick
}
// Вывод:
// ServerConfig {
//     hostname: "diag-node-01",
//     port: 8080,
//     debug: true,
//     modules: ["accel_diag", "nic_diag", "cpu_diag"],
//     level: Quick,
// }
// debug (default): false
// level (default): Quick
```

</details>

----
