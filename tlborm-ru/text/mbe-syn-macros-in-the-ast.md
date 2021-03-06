% Макросы в AST

Как раньше отмечалось, обработка макросов в Rust происходит *после* построения
AST. Поэтому синтаксис, используемый для вызова макроса, *должен быть*
собственной частью синтаксиса языка. На самом деле есть несколько форм
"расширений синтаксиса", которые являются частью синтаксиса Rust. Конкретно, это
следующие формы (с примерами):

* `# [ $arg ]`; *например*, `#[derive(Clone)]`, `#[no_mangle]`, …
* `# ! [ $arg ]`; *например*, `#![allow(dead_code)]`, `#![crate_name="blang"]`, …
* `$name ! $arg`; *например*, `println!("Hi!")`, `concat!("a", "b")`, …
* `$name ! $arg0 $arg1`; *например*, `macro_rules! dummy { () => {}; }`.

Первые две -  "атрибуты" и относятся к специальным конструкциям языка (таким как
`#[repr(C)]`, которые используются при запросе C-совместимого ABI для
пользовательского типа) и к расширениям синтаксиса (таким как
`#[derive(Clone)]`). В данный момент нет возможности определить макрос, который
бы использовал эти формы.

Третья форма представляет для нас интерес: это форма, доступная для
использования макросов. Помните, что ее использование *не ограничено* только
макросами: это общая форма расширения синтаксиса.  Например, если `format!` -
это макрос, то `format_args!` (который используется для *реализации* `format!`) -
*нет*.

Четвертая форма - это, по существу, вариация, которая *недоступна* для макросов.
*Единственное* место, на самом деле, где она используется, является
`macro_rules!`, к которому мы еще вернемся.

Если отбросить всё, кроме третьей формы (`$name ! $arg`), то возникает вопрос:
откуда парсер Rust знает, как выглядит `$arg` для всех возможных расширений
синтаксиса? Ответ заключается в том, что ему и *не нужно* это знать. Аргументом
расширения синтаксиса является *одиночное* дерево токенов. Говоря более
конкретно, это одиночное *безлистное* дерево токенов: `(...)`, `[...]` или
`{...}`. Зная это, становится ясно, как парсер может разобрать все эти формы
вызова:

```ignore
bitflags! {
    flags Color: u8 {
        const RED    = 0b0001,
        const GREEN  = 0b0010,
        const BLUE   = 0b0100,
        const BRIGHT = 0b1000,
    }
}

lazy_static! {
    static ref FIB_100: u32 = {
        fn fib(a: u32) -> u32 {
            match a {
                0 => 0,
                1 => 1,
                a => fib(a-1) + fib(a-2)
            }
        }

        fib(100)
    };
}

fn main() {
    let colors = vec![RED, GREEN, BLUE];
    println!("Hello, World!");
}
```

Хотя эти вызовы могут *выглядеть* так, как будто содержат разный код Rust,
парсер просто видит коллекцию бессмысленных деревьев токенов. Для ясности мы
можем заменить все эти синтаксические "черные ящики" на ⬚ и получим:

```text
bitflags! ⬚

lazy_static! ⬚

fn main() {
    let colors = vec! ⬚;
    println! ⬚;
}
```

Повторим: парсер *ничего* не делает с ⬚; он запоминает токены, которые они
содержат, но не пытается их *разобрать*.

Можно сделать важные выводы из этого:

* В Rust есть несколько типов расширения синтаксиса. Мы будем говорить *только*
о макросах, определяемых конструкцией `macro_rules!`.

* Только из-за того, что вы видите что-то в форме `$name! $arg`, не означает,
что это на самом деле макрос; это может быть другой тип расширения синтаксиса.

* На вход в каждый макрос подается одиночное безлистное дерево токенов.

* Макросы (расширения синтаксиса в общем смысле) разбираются как
часть абстрактного синтаксического дерева (AST).


> **Пояснение**: Из-за первого вывода некоторое описываемое ниже (включая
 следующий параграф) будет применяться к расширениям синтаксиса *в общем смысле*.
 [^writer-is-lazy]

[^writer-is-lazy]: Гораздо удобнее и быстрее писать "макрос", чем "расширение
синтаксиса".

Последний пункт является наиболее важным, поскольку он имеет *серьезные*
последствия. Из-за того, что макросы разбираются в AST, они могут появляться
**только** на тех позициях, в которых явно поддерживаются. Говоря
конкретней, они могут появляться на месте:

* Образцов в правилах самих макросов
* Утверждений
* Выражений
* Элементов
* Элементов `impl`

Некоторые вещи, *не* указанные в этом списке:

* Идентификаторы
* Варианты при поиске совпадения с образцом
* Поля структур
* Типы [^type-macros]

[^type-macros]: Макросы типа доступны в нестабильном Rust с
`#![feature(type_macros)]`; см. [Issue #27336](https://github.com/rust-lang/rust/issues/27336).

Нет совсем-совсем *никакой* возможности использовать макросы в месте, *не*
указанном в первом списке.
