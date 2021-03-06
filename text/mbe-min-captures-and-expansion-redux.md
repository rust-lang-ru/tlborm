% Вернемся к Метапеременным и Развертыванию

Когда парсер начинает захватывать токены в метапеременные, *он уже не может
остановиться или откатиться*. Это означает, что совпадение с образцом из второго
правила макроса, приведенного ниже, *никогда не произойдет*, что бы ни подавали
на вход:

```ignore
macro_rules! dead_rule {
    ($e:expr) => { ... };
    ($i:ident +) => { ... };
}
```

Представим что случится, если этот макрос вызвать как  `dead_rule!(x+)`.
Интерпретатор начнет с первого правила и попытается разобрать вход как
выражение. Первый токен (`x`) подходит в качестве выражения. Второй *тоже*,
представляя собой узел двоичного сложения.

В таком случае, подав это выражение без второго оператора сложения на вход, вы
ждете, что парсер пропустит первое правило и перейдет ко второму. Вместо этого
парсер вызовет панику и прервет компиляцию, ссылаясь на ошибку в синтаксисе.

Таким образом, нужно понимать, что при написании правил в образцах надо
указывать не какую-либо конкретную, а наиболее общую форму.

Для защиты от дальнейших изменений синтаксиса, затрагивающих интерпретацию входа
макроса, `macro_rules!` ограничивает то, что может идти после различных
метапеременных. Полный список ограничений для Rust 1.3 следующий:

* `item`: что угодно.
* `block`: что угодно.
* `stmt`: `=>` `,` `;`
* `pat`: `=>` `,` `=` `if` `in`
* `expr`: `=>` `,` `;`
* `ty`: `,` `=>` `:` `=` `>` `;` `as`
* `ident`: что угодно.
* `path`: `,` `=>` `:` `=` `>` `;` `as`
* `meta`: что угодно.
* `tt`: что угодно.

В дополнение к этому, `macro_rules!` обычно запрещает идти одному повторению
сразу за другим, даже  если их содержимое не конфликтует.

Еще одна особенность подстановки, которая обычно всех удивляет, это то, что
подстановка *не* основана на токенах, несмотря на то, что *выглядит* именно
так. Вот простая демонстрация этого:

```rust
macro_rules! capture_expr_then_stringify {
    ($e:expr) => {
        stringify!($e)
    };
}

fn main() {
    println!("{:?}", stringify!(dummy(2 * (1 + (3)))));
    println!("{:?}", capture_expr_then_stringify!(dummy(2 * (1 + (3)))));
}
```

Заметьте, что `stringify!` - это встроенное расширение синтаксиса, которое берет
все токены на входе и объединяет их в одну большую строку.

Вывод будет следующий:

```text
"dummy ( 2 * ( 1 + ( 3 ) ) )"
"dummy(2 * (1 + (3)))"
```

Заметьте, что выводы разные, *несмотря на* одинаковые входные данные.  Это
происходит из-за того, что первый вызов преобразует в строку выражение из
деревьев токенов, а второй преобразует в строку *узел выражения AST*.

Для визуализации разницы - вот, что на входе в макрос `stringify!` в первом
случае:

```text
«dummy» «(   )»
   ╭───────┴───────╮
    «2» «*» «(   )»
       ╭───────┴───────╮
        «1» «+» «(   )»
                 ╭─┴─╮
                  «3»
```

…а вот, с чем, он вызывается во втором:

```text
« »
 │ ┌─────────────┐
 └╴│ Call        │
   │ fn: dummy   │   ┌─────────┐
   │ args: ◌     │╶─╴│ BinOp   │
   └─────────────┘   │ op: Mul │
                   ┌╴│ lhs: ◌  │
        ┌────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
        │ LitInt │╶┘ └─────────┘ └╴│ BinOp   │
        │ val: 2 │                 │ op: Add │
        └────────┘               ┌╴│ lhs: ◌  │
                      ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────┐
                      │ LitInt │╶┘ └─────────┘ └╴│ LitInt │
                      │ val: 1 │                 │ val: 3 │
                      └────────┘                 └────────┘
```

Как вы можете заметить, здесь только *одно* дерево токенов, которое содержит
AST, разобранный после вызова `capture_expr_then_stringify!`. Поэтому то, что вы
видите в выводе - не преобразованные в строку токены, а преобразованные в строку
*узлы AST*.

Эта особенность имеет и другие следствия. Представьте следующее:

```rust
macro_rules! capture_then_match_tokens {
    ($e:expr) => {match_tokens!($e)};
}

macro_rules! match_tokens {
    ($a:tt + $b:tt) => {"got an addition"};
    (($i:ident)) => {"got an identifier"};
    ($($other:tt)*) => {"got something else"};
}

fn main() {
    println!("{}\n{}\n{}\n",
        match_tokens!((caravan)),
        match_tokens!(3 + 6),
        match_tokens!(5));
    println!("{}\n{}\n{}",
        capture_then_match_tokens!((caravan)),
        capture_then_match_tokens!(3 + 6),
        capture_then_match_tokens!(5));
}
```

Вывод:

```text
got an identifier
got an addition
got something else

got something else
got something else
got something else
```

После разбора входа в узлы AST, подставленный результат становиться
*неразрывным*; *т.е.* вы никогда больше не сможете проверить его содержимое или
найти совпадение с образцом внутри него.

Вот *еще* пример, который может особенно смутить:

```rust
macro_rules! capture_then_what_is {
    (#[$m:meta]) => {what_is!(#[$m])};
}

macro_rules! what_is {
    (#[no_mangle]) => {"no_mangle attribute"};
    (#[inline]) => {"inline attribute"};
    ($($tts:tt)*) => {concat!("something else (", stringify!($($tts)*), ")")};
}

fn main() {
    println!(
        "{}\n{}\n{}\n{}",
        what_is!(#[no_mangle]),
        what_is!(#[inline]),
        capture_then_what_is!(#[no_mangle]),
        capture_then_what_is!(#[inline]),
    );
}
```

Вывод:

```text
no_mangle attribute
inline attribute
something else (# [ no_mangle ])
something else (# [ inline ])
```

Единственный способ избежать этого - использовать захват в метапеременные,
применяя `tt` или `ident` типы. Единственное, что вы сможете сделать с
результатом, если вы будете использовать захват в метапеременные другого типа
- только передать его прямо на выход.
