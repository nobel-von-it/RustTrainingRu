### Небезопасный Rust (Unsafe Rust)

> **Чему вы научитесь:** Когда и как использовать небезопасный код (`unsafe`) — разыменование сырых указателей, интерфейс внешних функций FFI (Foreign Function Interface) для вызова C из Rust и наоборот, `CString`/`CStr` для взаимодействия строк, а также написание безопасных оболочек вокруг небезопасного кода.

- Ключевое слово `unsafe` открывает доступ к функциям, которые обычно запрещены компилятором Rust:
    - Разыменование сырых указателей
    - Доступ к *изменяемым* статическим переменным
    - https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
- С большой силой приходит большая ответственность:
    - Блок `unsafe` сообщает компилятору: «Я, программист, беру на себя ответственность за соблюдение инвариантов, которые обычно гарантирует компилятор».
    - Вы должны гарантировать отсутствие пересекающихся изменяемых и неизменяемых ссылок (aliased references), отсутствие висячих указателей, некорректных ссылок и т. д.
    - Использование `unsafe` должно быть ограничено как можно меньшей областью видимости.
    - Весь код, использующий `unsafe`, должен сопровождаться комментарием `// Safety:`, описывающим допущения.

### Примеры небезопасного Rust

```rust
unsafe fn harmless() {}
fn main() {
    // Safety: Мы вызываем безвредную небезопасную функцию
    unsafe {
        harmless();
    }
    let a = 42u32;
    let p = &a as *const u32;
    // Safety: p — валидный указатель на переменную, которая останется в области видимости
    unsafe {
        println!("{}", *p);
    }
    // Safety: Небезопасно; приведено только для иллюстрации
    let dangerous_buffer = 0xb8000 as *mut u32;
    unsafe {
        println!("About to go kaboom!!!");
        *dangerous_buffer = 0; // На большинстве современных систем это вызовет ошибку сегментации (SEGV)
    }
}
```

### Простой пример FFI (функция библиотеки Rust, используемая в C)

## Строки в FFI: CString и CStr

FFI (Foreign Function Interface) — это механизм, который Rust использует для вызова функций, написанных на других языках (таких как C), и наоборот.

При взаимодействии с кодом на C типы строк Rust `String` и `&str` (которые представляют собой UTF-8 без завершающего нулевого символа) несовместимы напрямую со строками C (которые являются массивами байтов, заканчивающимися нулем). Для этой цели Rust предоставляет типы `CString` (владеющий) и `CStr` (заимствованный) из модуля `std::ffi`:

| Тип | Аналог | Когда использовать |
|------|-------------|----------|
| `CString` | `String` (владеющий) | Создание строки C из данных Rust |
| `&CStr` | `&str` (заимствованный) | Получение строки C из внешнего кода |

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

fn demo_ffi_strings() {
    // Создание C-совместимой строки (добавляет завершающий нуль)
    let c_string = CString::new("Hello from Rust").expect("CString::new failed");
    let ptr: *const c_char = c_string.as_ptr();

    // Преобразование строки C обратно в Rust (небезопасно, так как мы доверяем указателю)
    // Safety: ptr валиден и заканчивается нулем (мы только что создали его выше)
    let back_to_rust: &CStr = unsafe { CStr::from_ptr(ptr) };
    let rust_str: &str = back_to_rust.to_str().expect("Invalid UTF-8");
    println!("{}", rust_str);
}
```

> **Внимание**: Метод `CString::new()` вернет ошибку, если входные данные содержат внутренние нулевые байты (`\0`). Всегда обрабатывайте результат `Result`. Ниже вы увидите широкое использование `CStr` в примерах с FFI.

- Методы `FFI` должны быть помечены атрибутом `#[no_mangle]`, чтобы компилятор не искажал (mangled) их имена.
- Мы скомпилируем крейт как статическую библиотеку.
    ```rust
    #[no_mangle] 
    pub extern "C" fn add(left: u64, right: u64) -> u64 {
        left + right
    }
    ```
- Мы скомпилируем следующий код на C и свяжем его с нашей статической библиотекой.
    ```c
    #include <stdio.h>
    #include <stdint.h>
    extern uint64_t add(uint64_t, uint64_t);
    int main() {
        printf("Add returned %llu\n", add(21, 21));
    }
    ``` 

### Сложный пример FFI
- В следующих примерах мы создадим интерфейс логирования на Rust и предоставим к нему доступ для Python и C.
    - Мы увидим, как один и тот же интерфейс можно нативно использовать из Rust и C.
    - Мы изучим использование инструментов вроде `cbindgen` для генерации заголовочных файлов C.
    - Мы посмотрим, как небезопасные оболочки (`unsafe` wrappers) могут служить мостом к безопасному коду на Rust.

## Вспомогательные функции логгера
```rust
fn create_or_open_log_file(log_file: &str, overwrite: bool) -> Result<File, String> {
    if overwrite {
        File::create(log_file).map_err(|e| e.to_string())
    } else {
        OpenOptions::new()
            .write(true)
            .append(true)
            .open(log_file)
            .map_err(|e| e.to_string())
    }
}

fn log_to_file(file_handle: &mut File, message: &str) -> Result<(), String> {
    file_handle
        .write_all(message.as_bytes())
        .map_err(|e| e.to_string())
}
```

## Структура логгера
```rust
struct SimpleLogger {
    log_level: LogLevel,
    file_handle: File,
}

impl SimpleLogger {
    fn new(log_file: &str, overwrite: bool, log_level: LogLevel) -> Result<Self, String> {
        let file_handle = create_or_open_log_file(log_file, overwrite)?;
        Ok(Self {
            file_handle,
            log_level,
        })
    }

    fn log_message(&mut self, log_level: LogLevel, message: &str) -> Result<(), String> {
        if log_level as u32 <= self.log_level as u32 {
            let timestamp = Local::now().format("%Y-%m-%d %H:%M:%S").to_string();
            let message = format!("Simple: {timestamp} {log_level} {message}\n");
            log_to_file(&mut self.file_handle, &message)
        } else {
            Ok(())
        }
    }
}
```

## Тестирование
- Тестирование функциональности в Rust тривиально.
    - Тестовые методы помечаются атрибутом `#[test]` и не включаются в составной скомпилированный бинарный файл.
    - Для целей тестирования легко создавать мок-методы.
```rust
#[test]
fn testfunc() -> Result<(), String> {
    let mut logger = SimpleLogger::new("test.log", false, LogLevel::INFO)?;
    logger.log_message(LogLevel::TRACELEVEL1, "Hello world")?;
    logger.log_message(LogLevel::CRITICAL, "Critical message")?;
    Ok(()) // Компилятор автоматически удаляет (drops) logger в этом месте
}
```
```bash
cargo test
```

## FFI между C и Rust
- cbindgen — отличный инструмент для генерации заголовочных файлов для экспортируемых функций Rust.
    - Ее можно установить с помощью cargo.
```bash
cargo install cbindgen
cbindgen 
```
- Функции и структуры могут экспортироваться с помощью атрибутов `#[no_mangle]` и `#[repr(C)]`.
    - Мы примем стандартный шаблон интерфейса, передавая указатель `**` на саму реализацию и возвращая 0 при успешном завершении и ненулевое значение при ошибке.
    - **Непрозрачные (opaque) против прозрачных структур**: Наш `SimpleLogger` передается как *непрозрачный указатель* (`*mut SimpleLogger`) — код на C никогда не обращается к его полям напрямую, поэтому `#[repr(C)]` **не** требуется. Используйте `#[repr(C)]`, когда коду на C необходимо читать или записывать поля структуры напрямую:

```rust
// Непрозрачная (Opaque) — C только хранит указатель, никогда не считывает поля. #[repr(C)] не требуется.
struct SimpleLogger { /* Rust-only fields */ }

// Прозрачная (Transparent) — C считывает/записывает поля напрямую. ОБЯЗАТЕЛЬНО использовать #[repr(C)].
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
```
```c
typedef struct SimpleLogger SimpleLogger;
uint32_t create_simple_logger(const char *file_name, struct SimpleLogger **out_logger);
uint32_t log_entry(struct SimpleLogger *logger, const char *message);
uint32_t drop_logger(struct SimpleLogger *logger);
```

- Обратите внимание, что нам нужно выполнить много проверок на корректность.
- Мы должны явно вызвать утечку памяти (leak), чтобы предотвратить автоматическое освобождение памяти со стороны Rust.
```rust
#[no_mangle] 
pub extern "C" fn create_simple_logger(file_name: *const std::os::raw::c_char, out_logger: *mut *mut SimpleLogger) -> u32 {
    use std::ffi::CStr;
    // Убедимся, что указатель не NULL
    if file_name.is_null() || out_logger.is_null() {
        return 1;
    }
    // Safety: Переданный указатель по контракту является либо NULL, либо оканчивается нулем
    let file_name = unsafe {
        CStr::from_ptr(file_name)
    };
    let file_name = file_name.to_str();
    // Убедимся, что file_name не содержит некорректных символов
    if file_name.is_err() {
        return 1;
    }
    let file_name = file_name.unwrap();
    // Примем некоторые настройки по умолчанию; в реальной жизни мы бы передавали их
    let new_logger = SimpleLogger::new(file_name, true, LogLevel::CRITICAL);
    // Проверим, удалось ли нам создать логгер
    if new_logger.is_err() {
        return 1;
    }
    let new_logger = Box::new(new_logger.unwrap());
    // Это предотвращает удаление Box при выходе из области видимости
    let logger_ptr: *mut SimpleLogger = Box::leak(new_logger);
    // Safety: logger не равен null и logger_ptr валиден
    unsafe {
        *out_logger = logger_ptr;
    }
    return 0;
}
```

- Похожие проверки на ошибки мы делаем в функции `log_entry()`
```rust
#[no_mangle]
pub extern "C" fn log_entry(logger: *mut SimpleLogger, message: *const std::os::raw::c_char) -> u32 {
    use std::ffi::CStr;
    if message.is_null() || logger.is_null() {
        return 1;
    }
    // Safety: message не равен null
    let message = unsafe {
        CStr::from_ptr(message)
    };
    let message = message.to_str();
    // Убедимся, что имя файла не содержит некорректных символов
    if message.is_err() {
        return 1;
    }
    // Safety: logger — валидный указатель, ранее созданный с помощью create_simple_logger()
    unsafe {
        (*logger).log_message(LogLevel::CRITICAL, message.unwrap()).is_err() as u32
    }
}

#[no_mangle]
pub extern "C" fn drop_logger(logger: *mut SimpleLogger) -> u32 {
    if logger.is_null() {
        return 1;
    }
    // Safety: logger — валидный указатель, ранее созданный с помощью create_simple_logger()
    unsafe {
        // Это конструирует Box<SimpleLogger>, который удаляется при выходе из области видимости
        let _ = Box::from_raw(logger);
    }
    0
}
```

- Мы можем протестировать наше (C)-FFI с помощью Rust или написав программу на C
```rust
#[test]
fn test_c_logger() {
    // Префикс c".." создает строку, оканчивающуюся нулем (null-terminated)
    let file_name = c"test.log".as_ptr() as *const std::os::raw::c_char;
    let mut c_logger: *mut SimpleLogger = std::ptr::null_mut();
    assert_eq!(create_simple_logger(file_name, &mut c_logger), 0);
    // Это ручной способ создания строк c"..."
    let message = b"message from C\0".as_ptr() as *const std::os::raw::c_char;
    assert_eq!(log_entry(c_logger, message), 0);
    drop_logger(c_logger);
}
```
```c
#include "logger.h"
...
int main() {
    SimpleLogger *logger = NULL;
    if (create_simple_logger("test.log", &logger) == 0) {
        log_entry(logger, "Hello from C");
        drop_logger(logger); /*Необходимо для закрытия дескриптора файла и т.д.*/
    } 
    ...
}
```

## Обеспечение корректности небезопасного кода
- Краткий вывод (TL;DR): использование `unsafe` требует осознанного подхода.
    - Всегда документируйте предположения о безопасности, сделанные в коде, и проверяйте их со специалистами.
    - Используйте инструменты вроде cbindgen, Miri, Valgrind, которые могут помочь подтвердить корректность.
    - **Никогда не позволяйте панике раскручиваться через границу FFI** — это приводит к неопределенному поведению (UB). Используйте `std::panic::catch_unwind` на точках входа FFI или настройте `panic = "abort"` в профиле вашего проекта.
    - Если структура разделяется через FFI, пометьте ее `#[repr(C)]`, чтобы гарантировать C-совместимое расположение данных в памяти.
    - Изучите https://doc.rust-lang.org/nomicon/intro.html («Rustonomicon» — темные искусства небезопасного Rust).
    - Пользуйтесь помощью внутренних экспертов.

### Инструменты верификации: Miri против Valgrind

Разработчики C++ знакомы с Valgrind и санитайзерами. В Rust есть всё это **плюс** Miri, который гораздо точнее выявляет UB, специфичное для Rust:

| | **Miri** | **Valgrind** | **C++ санитайзеры (ASan/MSan/UBSan)** |
|---|---------|-------------|--------------------------------------|
| **Что находит** | Специфичное для Rust UB: модель stacked borrows, недопустимые дискриминанты enum, чтение неинициализированной памяти, нарушения наложения ссылок | Утечки памяти, использование после освобождения (use-after-free), некорректные чтение/запись, неинициализированная память | Переполнение буфера, использование после освобождения, состояние гонки, UB |
| **Как работает** | Интерпретирует MIR (среднеуровневое промежуточное представление Rust) — без нативного выполнения | Инструментирует скомпилированный бинарный файл во время выполнения | Инструментирование во время компиляции |
| **Поддержка FFI** | ❌ Не может переходить границу FFI (пропускает вызовы C) | ✅ Работает с любым скомпилированным бинарником, включая FFI | ✅ Работает, если код C также скомпилирован с санитайзерами |
| **Скорость** | Примерно в 100 раз медленнее нативного | Примерно в 10-50 раз медленнее | Примерно в 2-5 раз медленнее |
| **Когда использовать** | Чистый небезопасный код Rust, инварианты структур данных | Код FFI, интеграционные тесты всего бинарного файла | Сторона C/C++ в FFI, тестирование критических по производительности мест |
| **Находит ошибки наложения ссылок (aliasing)** | ✅ Модель Stacked Borrows | ❌ | Частично (TSan для состояний гонки за данные) |

**Рекомендация**: Используйте **оба** инструмента — Miri для чистого небезопасного Rust, Valgrind для интеграции с FFI:

- **Miri** — находит специфическое для Rust UB, которое Valgrind не видит (нарушения правил наложения ссылок, невалидные значения enum, нарушения модели stacked borrows):
    ```bash
    rustup +nightly component add miri
    cargo +nightly miri test                    # Запуск всех тестов под Miri
    cargo +nightly miri test -- test_name       # Запуск конкретного теста
    ```
    > ⚠️ Miri требует nightly-версии компилятора и не может выполнять вызовы FFI. Изолируйте небезопасную логику Rust в отдельные тестируемые модули.

- **Valgrind** — уже знакомый вам инструмент, работает со скомпилированным бинарным файлом, включая FFI:
    ```bash
    sudo apt install valgrind
    cargo install cargo-valgrind
    cargo valgrind test                         # Запуск всех тестов под Valgrind
    ```
    > Помогает находить утечки в паттернах `Box::leak` / `Box::from_raw`, часто встречающихся в коде FFI.

- **cargo-careful** — запускает тесты с включенными дополнительными проверками во время выполнения (промежуточный вариант между обычными тестами и Miri):
    ```bash
    cargo install cargo-careful
    cargo +nightly careful test
    ```

## Резюме по небезопасному Rust (Unsafe Rust)
- `cbindgen` — отличный инструмент для реализации C FFI к Rust.
    - Используйте `bindgen` для FFI-интерфейсов в обратном направлении (обратитесь к его подробной документации).
- **Не думайте, что ваш небезопасный код корректен или что его можно безопасно использовать из безопасного Rust. Допустить ошибки здесь очень легко, и даже код, который с виду работает правильно, может содержать скрытые проблемы.**
    - Используйте специальные инструменты для проверки корректности.
    - Если у вас остались сомнения, обратитесь за советом к экспертам.
- Обязательно снабжайте код `unsafe` комментариями с явным документированием предположений и объяснением того, почему этот код корректен.
    - Вызывающие небезопасный код методы также должны иметь соответствующие комментарии о безопасности и соблюдать ограничения.

# Упражнение: Написание безопасной оболочки FFI

🔴 **Сложная задача** — требует понимания блоков unsafe, сырых указателей и разработки безопасного API

- Напишите безопасную оболочку на Rust вокруг небезопасной функции в стиле FFI. Это упражнение имитирует вызов функции C, которая записывает отформатированную строку в предоставленный вызывающей стороной буфер.
- **Шаг 1**: Реализуйте небезопасную функцию `unsafe_greet`, которая записывает приветствие в сырой буфер `*mut u8`.
- **Шаг 2**: Напишите безопасную оболочку `safe_greet`, которая выделяет буфер `Vec<u8>`, вызывает небезопасную функцию и возвращает строку `String`.
- **Шаг 3**: Добавьте корректные комментарии `// Safety:` к каждому блоку `unsafe`.

**Начальный код:**
```rust
use std::fmt::Write as _;

/// Имитирует функцию C: записывает "Hello, <name>!" в буфер.
/// Возвращает количество записанных байтов (исключая завершающий нуль).
/// # Safety
/// - `buf` должен указывать как минимум на `buf_len` байт, доступных для записи
/// - `name` должен быть валидным указателем на строку C, оканчивающуюся нулем
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // TODO: Сформируйте приветствие, скопируйте байты в buf, верните длину
    // Подсказка: используйте std::ffi::CStr::from_ptr или перебирайте байты вручную
    todo!()
}

/// Безопасная оболочка — в публичном API нет unsafe
fn safe_greet(name: &str) -> Result<String, String> {
    // TODO: Выделите буфер Vec<u8>, создайте имя, оканчивающееся нулем,
    // вызовите unsafe_greet внутри блока unsafe с комментарием Safety,
    // преобразуйте результат обратно в String
    todo!()
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("Error: {e}"),
    }
    // Ожидаемый вывод: Hello, Rustacean!
}
```

<details><summary>Решение (нажмите, чтобы раскрыть)</summary>

```rust
use std::ffi::CStr;

/// Имитирует функцию C: записывает "Hello, <name>!" в буфер.
/// Возвращает количество записанных байтов или -1, если буфер слишком мал.
/// # Safety
/// - `buf` должен указывать как минимум на `buf_len` байт, доступных для записи
/// - `name` должен быть валидным указателем на строку C, оканчивающуюся нулем
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // Safety: вызывающая сторона гарантирует, что name — валидная строка, оканчивающаяся нулем
    let name_cstr = unsafe { CStr::from_ptr(name as *const std::os::raw::c_char) };
    let name_str = match name_cstr.to_str() {
        Ok(s) => s,
        Err(_) => return -1,
    };
    let greeting = format!("Hello, {}!", name_str);
    if greeting.len() > buf_len {
        return -1;
    }
    // Safety: buf указывает как минимум на buf_len байт, доступных для записи (гарантия вызывающей стороны)
    unsafe {
        std::ptr::copy_nonoverlapping(greeting.as_ptr(), buf, greeting.len());
    }
    greeting.len() as isize
}

/// Безопасная оболочка — в публичном API нет unsafe
fn safe_greet(name: &str) -> Result<String, String> {
    let mut buffer = vec![0u8; 256];
    // Создаем версию имени, оканчивающуюся нулем, для API на C
    let name_with_null: Vec<u8> = name.bytes().chain(std::iter::once(0)).collect();

    // Safety: buffer содержит 256 байт для записи, name_with_null оканчивается нулем
    let bytes_written = unsafe {
        unsafe_greet(buffer.as_mut_ptr(), buffer.len(), name_with_null.as_ptr())
    };

    if bytes_written < 0 {
        return Err("Buffer too small or invalid name".to_string());
    }

    String::from_utf8(buffer[..bytes_written as usize].to_vec())
        .map_err(|e| format!("Invalid UTF-8: {e}"))
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("Error: {e}"),
    }
}
// Вывод:
// Hello, Rustacean!
```

</details>

----
