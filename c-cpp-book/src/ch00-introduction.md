# Курс быстрого старта по Rust для программистов на C/C++

## Обзор курса
- Обзор курса
    - Аргументы в пользу Rust (с точки зрения как C, так и C++)
    - Локальная установка
    - Типы, функции, управляющие конструкции, сопоставление с образцом
    - Модули, cargo
    - Типажи, обобщения
    - Коллекции, обработка ошибок
    - Замыкания, управление памятью, времена жизни, умные указатели
    - Конкурентность
    - Небезопасный Rust (Unsafe Rust), включая интерфейс внешних функций (FFI)
    - Основы `no_std` и встраиваемого Rust для разработчиков прошивок
    - Практические примеры: паттерны переноса кода с C++ на Rust в реальных проектах
- В этом курсе мы не будем рассматривать асинхронный (`async`) Rust — обратитесь к сопутствующему руководству [Async Rust Training](../async-book/) для подробного изучения футуров (futures), исполнителей (executors), `Pin`, tokio и шаблонов асинхронного программирования в продакшене

---

# Руководство для самостоятельного изучения

Этот материал подходит как для занятий с преподавателем, так и для самостоятельного изучения. Если вы проходите его сами, вот как получить от него максимум пользы:

**Рекомендации по темпу:**

| Главы | Тема | Рекомендуемое время | Контрольная точка |
|-------|------|--------------------|-------------------|
| 1–4 | Настройка, типы, управление потоком | 1 день | Вы можете написать консольный конвертер температур |
| 5–7 | Структуры данных, владение | 1–2 дня | Вы можете объяснить, *почему* `let s2 = s1` делает `s1` недействительным |
| 8–9 | Модули, обработка ошибок | 1 день | Вы можете создать многофайловый проект, передающий ошибки с помощью `?` |
| 10–12 | Типажи, обобщения, замыкания | 1–2 дня | Вы можете написать обобщенную функцию с ограничениями типажей |
| 13–14 | Конкурентность, unsafe/FFI | 1 день | Вы можете написать потокобезопасный счетчик с использованием `Arc<Mutex<T>>` |
| 15–16 | Глубокое погружение | В своем темпе | Справочный материал — читайте по мере необходимости |
| 17–19 | Лучшие практики и справочники | В своем темпе | Обращайтесь к материалу при написании реального кода |

**Как работать с упражнениями:**
- В каждой главе есть практические упражнения с указанием сложности: 🟢 Начальный, 🟡 Средний, 🔴 Сложный
- **Всегда пытайтесь выполнить упражнение, прежде чем открывать решение.** Борьба с проверкой заимствований (borrow checker) — это часть обучения. Сообщения об ошибках компилятора — ваши лучшие учителя.
- Если вы зависли более чем на 15 минут, откройте решение, изучите его, затем закройте и попробуйте написать все с нуля.
- [Rust Playground](https://play.rust-lang.org/) позволяет запускать код без локальной установки.

**Если вы зашли в тупик:**
- Внимательно прочитайте сообщение об ошибке компилятора — сообщения об ошибках в Rust невероятно информативны.
- Перечитайте соответствующий раздел; такие концепции, как владение (глава 7), часто становятся понятны со второго раза.
- [Документация стандартной библиотеки Rust](https://doc.rust-lang.org/std/) превосходна — ищите там любые типы или методы.
- Для изучения асинхронных паттернов обратитесь к сопутствующему руководству [Async Rust Training](../async-book/).

---

# Оглавление

## Часть I — Основы

### 1. Введение и мотивация
- [Знакомство с докладчиком и общий подход](ch01-introduction-and-motivation.md#speaker-intro-and-general-approach)
- [Аргументы в пользу Rust](ch01-introduction-and-motivation.md#the-case-for-rust)
- [Как Rust решает эти проблемы?](ch01-introduction-and-motivation.md#how-does-rust-address-these-issues)
- [Другие особенности и преимущества Rust](ch01-introduction-and-motivation.md#other-rust-usps-and-features)
- [Краткий справочник: Rust в сравнении с C/C++](ch01-introduction-and-motivation.md#quick-reference-rust-vs-cc)
- [Почему разработчикам C/C++ нужен Rust](ch01-1-why-c-cpp-developers-need-rust.md)
  - [Что устраняет Rust — полный список](ch01-1-why-c-cpp-developers-need-rust.md#what-rust-eliminates--the-complete-list)
  - [Общие проблемы C и C++](ch01-1-why-c-cpp-developers-need-rust.md#the-problems-shared-by-c-and-c)
  - [Дополнительные проблемы, привносимые C++](ch01-1-why-c-cpp-developers-need-rust.md#c-adds-more-problems-on-top)
  - [Как Rust решает всё это](ch01-1-why-c-cpp-developers-need-rust.md#how-rust-addresses-all-of-this)

### 2. Начало работы
- [Достаточно разговоров: покажите мне код](ch02-getting-started.md#enough-talk-already-show-me-some-code)
- [Локальная установка Rust](ch02-getting-started.md#rust-local-installation)
- [Пакеты Rust (crates)](ch02-getting-started.md#rust-packages-crates)
- [Пример: cargo и пакеты](ch02-getting-started.md#example-cargo-and-crates)

### 3. Базовые типы и переменные
- [Встроенные типы Rust](ch03-built-in-types.md#built-in-rust-types)
- [Спецификация типов и присваивание в Rust](ch03-built-in-types.md#rust-type-specification-and-assignment)
- [Указание типов и выведение типов в Rust](ch03-built-in-types.md#rust-type-specification-and-inference)
- [Переменные и мутабельность в Rust](ch03-built-in-types.md#rust-variables-and-mutability)

### 4. Управляющие конструкции
- [Ключевое слово if в Rust](ch04-control-flow.md#rust-if-keyword)
- [Циклы в Rust с использованием while и for](ch04-control-flow.md#rust-loops-using-while-and-for)
- [Циклы в Rust с использованием loop](ch04-control-flow.md#rust-loops-using-loop)
- [Блоки выражений в Rust](ch04-control-flow.md#rust-expression-blocks)

### 5. Структуры данных и коллекции
- [Тип массива в Rust](ch05-data-structures.md#rust-array-type)
- [Кортежи в Rust](ch05-data-structures.md#rust-tuples)
- [Ссылки в Rust](ch05-data-structures.md#rust-references)
- [Ссылки в C++ и ссылки в Rust — ключевые отличия](ch05-data-structures.md#c-references-vs-rust-references--key-differences)
- [Срезы (slices) в Rust](ch05-data-structures.md#rust-slices)
- [Константы и статические переменные в Rust](ch05-data-structures.md#rust-constants-and-statics)
- [Строки в Rust: String против &str](ch05-data-structures.md#rust-strings-string-vs-str)
- [Структуры в Rust](ch05-data-structures.md#rust-structs)
- [Тип Vec\<T\> в Rust](ch05-data-structures.md#rust-vec-type)
- [Rust HashMap](ch05-data-structures.md#rust-hashmap-type)
- [Упражнение: Vec и HashMap](ch05-data-structures.md#exercise-vec-and-hashmap)

### 6. Сопоставление с образцом и перечисления
- [Типы перечислений в Rust](ch06-enums-and-pattern-matching.md#rust-enum-types)
- [Конструкция match в Rust](ch06-enums-and-pattern-matching.md#rust-match-statement)
- [Упражнение: Реализация сложения и вычитания с использованием match и enum](ch06-enums-and-pattern-matching.md#exercise-implement-add-and-subtract-using-match-and-enum)

### 7. Владение и управление памятью
- [Управление памятью в Rust](ch07-ownership-and-borrowing.md#rust-memory-management)
- [Владение, заимствование и времена жизни в Rust](ch07-ownership-and-borrowing.md#rust-ownership-borrowing-and-lifetimes)
- [Семантика перемещения в Rust](ch07-ownership-and-borrowing.md#rust-move-semantics)
- [Типаж Clone в Rust](ch07-ownership-and-borrowing.md#rust-clone)
- [Типаж Copy в Rust](ch07-ownership-and-borrowing.md#rust-copy-trait)
- [Типаж Drop в Rust](ch07-ownership-and-borrowing.md#rust-drop-trait)
- [Упражнение: перемещение (Move), копирование (Copy) и высвобождение (Drop)](ch07-ownership-and-borrowing.md#exercise-move-copy-and-drop)
- [Времена жизни и заимствование в Rust](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-and-borrowing)
- [Аннотации времени жизни в Rust](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-annotations)
- [Упражнение: Хранилище срезов с временами жизни](ch07-1-lifetimes-and-borrowing-deep-dive.md#exercise-slice-storage-with-lifetimes)
- [Глубокое погружение в правила элизии времен жизни](ch07-1-lifetimes-and-borrowing-deep-dive.md#lifetime-elision-rules-deep-dive)
- [Rust Box\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#rust-boxt)
- [Внутренняя мутабельность: Cell\<T\> и RefCell\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#interior-mutability-cellt-and-refcellt)
- [Разделяемое владение: Rc\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#shared-ownership-rct)
- [Упражнение: Разделяемое владение и внутренняя мутабельность](ch07-2-smart-pointers-and-interior-mutability.md#exercise-shared-ownership-and-interior-mutability)

### 8. Модули и пакеты
- [Пакеты и модули в Rust](ch08-crates-and-modules.md#rust-crates-and-modules)
- [Упражнение: Модули и функции](ch08-crates-and-modules.md#exercise-modules-and-functions)
- [Рабочие пространства (workspaces) и пакеты (crates)](ch08-crates-and-modules.md#workspaces-and-crates-packages)
- [Упражнение: Использование рабочих пространств и зависимостей пакетов](ch08-crates-and-modules.md#exercise-using-workspaces-and-package-dependencies)
- [Использование сторонних пакетов из crates.io](ch08-crates-and-modules.md#using-community-crates-from-cratesio)
- [Зависимости пакетов и SemVer](ch08-crates-and-modules.md#crates-dependencies-and-semver)
- [Упражнение: Использование пакета rand](ch08-crates-and-modules.md#exercise-using-the-rand-crate)
- [Cargo.toml и Cargo.lock](ch08-crates-and-modules.md#cargotoml-and-cargolock)
- [Возможности cargo test](ch08-crates-and-modules.md#cargo-test-feature)
- [Другие возможности Cargo](ch08-crates-and-modules.md#other-cargo-features)
- [Шаблоны тестирования](ch08-1-testing-patterns.md)

### 9. Обработка ошибок
- [Связь перечислений с Option и Result](ch09-error-handling.md#connecting-enums-to-option-and-result)
- [Тип Option в Rust](ch09-error-handling.md#rust-option-type)
- [Тип Result в Rust](ch09-error-handling.md#rust-result-type)
- [Упражнение: Реализация функции log() с использованием Option](ch09-error-handling.md#exercise-log-function-implementation-with-option)
- [Обработка ошибок в Rust](ch09-error-handling.md#rust-error-handling)
- [Упражнение: обработка ошибок](ch09-error-handling.md#exercise-error-handling)
- [Лучшие практики обработки ошибок](ch09-1-error-handling-best-practices.md)

### 10. Типажи и обобщения
- [Типажи в Rust](ch10-traits.md#rust-traits)
- [Перегрузка операторов в C++ → Типажи std::ops в Rust](ch10-traits.md#c-operator-overloading--rust-stdops-traits)
- [Упражнение: Реализация типажа Logger](ch10-traits.md#exercise-logger-trait-implementation)
- [Когда использовать enum, а когда dyn Trait](ch10-traits.md#when-to-use-enum-vs-dyn-trait)
- [Упражнение: Думайте, прежде чем переводить](ch10-traits.md#exercise-think-before-you-translate)
- [Обобщения (generics) в Rust](ch10-1-generics.md#rust-generics)
- [Упражнение: Обобщения](ch10-1-generics.md#exercise-generics)
- [Сочетание типажей и обобщений в Rust](ch10-1-generics.md#combining-rust-traits-and-generics)
- [Ограничения типажей для типов данных в Rust](ch10-1-generics.md#rust-traits-constraints-in-data-types)
- [Упражнение: Ограничения типажей и обобщения](ch10-1-generics.md#exercise-traits-constraints-and-generics)
- [Шаблон проектирования Type State и обобщения в Rust](ch10-1-generics.md#rust-type-state-pattern-and-generics)
- [Шаблон Строитель (Builder) в Rust](ch10-1-generics.md#rust-builder-pattern)

### 11. Дополнительные возможности системы типов
- [Типажи From и Into в Rust](ch11-from-and-into-traits.md#rust-from-and-into-traits)
- [Упражнение: From и Into](ch11-from-and-into-traits.md#exercise-from-and-into)
- [Типаж Default в Rust](ch11-from-and-into-traits.md#rust-default-trait)
- [Другие преобразования типов в Rust](ch11-from-and-into-traits.md#other-rust-type-conversions)

### 12. Функциональное программирование
- [Замыкания в Rust](ch12-closures.md#rust-closures)
- [Упражнение: Замыкания и захват переменных](ch12-closures.md#exercise-closures-and-capturing)
- [Итераторы в Rust](ch12-closures.md#rust-iterators)
- [Упражнение: Итераторы в Rust](ch12-closures.md#exercise-rust-iterators)
- [Справочник по мощным инструментам итераторов](ch12-1-iterator-power-tools.md#iterator-power-tools-reference)

### 13. Конкурентность
- [Конкурентность в Rust](ch13-concurrency.md#rust-concurrency)
- [Почему Rust предотвращает состояние гонки данных: Send и Sync](ch13-concurrency.md#why-rust-prevents-data-races-send-and-sync)
- [Упражнение: Многопоточный подсчет слов](ch13-concurrency.md#exercise-multi-threaded-word-count)

### 14. Небезопасный Rust и FFI
- [Небезопасный Rust (Unsafe Rust)](ch14-unsafe-rust-and-ffi.md#unsafe-rust)
- [Простой пример FFI](ch14-unsafe-rust-and-ffi.md#simple-ffi-example-rust-library-function-consumed-by-c)
- [Сложный пример FFI](ch14-unsafe-rust-and-ffi.md#complex-ffi-example)
- [Обеспечение корректности небезопасного кода](ch14-unsafe-rust-and-ffi.md#ensuring-correctness-of-unsafe-code)
- [Упражнение: Написание безопасной обертки для FFI](ch14-unsafe-rust-and-ffi.md#exercise-writing-a-safe-ffi-wrapper)

## Часть II — Глубокое погружение

### 15. no_std — Rust для «голого железа»
- [Что такое no_std?](ch15-no_std-rust-without-the-standard-library.md#what-is-no_std)
- [Когда использовать no_std против std](ch15-no_std-rust-without-the-standard-library.md#when-to-use-no_std-vs-std)
- [Упражнение: кольцевой буфер в no_std](ch15-no_std-rust-without-the-standard-library.md#exercise-no_std-ring-buffer)
- [Глубокое погружение во встраиваемые системы](ch15-1-embedded-deep-dive.md)

### 16. Практические примеры: перенос кода с C++ на Rust в реальном мире
- [Пример 1: Иерархия наследования → Диспетчеризация через перечисления](ch16-case-studies.md#case-study-1-inheritance-hierarchy--enum-dispatch)
- [Пример 2: Дерево shared_ptr → Шаблон Арена/индекс](ch16-case-studies.md#case-study-2-shared_ptr-tree--arenaindex-pattern)
- [Пример 3: Взаимодействие в фреймворке → Заимствование с временами жизни](ch16-1-case-study-lifetime-borrowing.md#case-study-3-framework-communication--lifetime-borrowing)
- [Пример 4: Божественный объект → Компонуемое состояние](ch16-1-case-study-lifetime-borrowing.md#case-study-4-god-object--composable-state)
- [Пример 5: Объекты типажей — когда они действительно нужны](ch16-1-case-study-lifetime-borrowing.md#case-study-5-trait-objects--when-they-are-right)

## Часть III — Лучшие практики и справочные материалы

### 17. Лучшие практики
- [Краткий обзор лучших практик в Rust](ch17-best-practices.md#rust-best-practices-summary)
- [Избегание избыточного clone()](ch17-1-avoiding-excessive-clone.md#avoiding-excessive-clone)
- [Избегание непроверяемой индексации](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing)
- [Упрощение пирамид присваивания](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids)
- [Итоговое упражнение: Конвейер диагностических событий](ch17-3-collapsing-assignment-pyramids.md#capstone-exercise-diagnostic-event-pipeline)
- [Экосистема логирования и трассировки](ch17-4-logging-and-tracing-ecosystem.md#logging-and-tracing-ecosystem)

### 18. Семантическое сравнение C++ и Rust
- [Приведение типов, препроцессор, модули, volatile, static, constexpr, SFINAE и многое другое](ch18-cpp-rust-semantic-deep-dives.md)

### 19. Макросы в Rust
- [Декларативные макросы (`macro_rules!`)](ch19-macros.md#declarative-macros-with-macro_rules)
- [Распространенные макросы стандартной библиотеки](ch19-macros.md#common-standard-library-macros)
- [Макросы derive](ch19-macros.md#derive-macros)
- [Атрибутные макросы](ch19-macros.md#attribute-macros)
- [Процедурные макросы](ch19-macros.md#procedural-macros-conceptual-overview)
- [Когда и что использовать: макросы против функций и обобщений](ch19-macros.md#when-to-use-what-macros-vs-functions-vs-generics)
- [Упражнения](ch19-macros.md#exercises)
