% Спихиваемые накопления

```rust
macro_rules! init_array {
    (@accum (0, $_e:expr) -> ($($body:tt)*))
        => {init_array!(@as_expr [$($body)*])};
    (@accum (1, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (0, $e) -> ($($body)* $e,))};
    (@accum (2, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (1, $e) -> ($($body)* $e,))};
    (@accum (3, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (2, $e) -> ($($body)* $e,))};
    (@as_expr $e:expr) => {$e};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            init_array!(@accum ($n, e.clone()) -> ())
        }
    };
}

let strings: [String; 3] = init_array![String::from("hi!"); 3];
# assert_eq!(format!("{:?}", strings), "[\"hi!\", \"hi!\", \"hi!\"]");
```

Все макросы в Rust **должны** в результате разворачиваться в законченный,
подходящий по синтаксису элемент (такой как выражение, элемент *и т.д.*). Это
означает, что нельзя сделать макрос, разворачивающийся в незаконченную
конструкцию.

Можно было бы надеяться, что пример выше можно выразить так:

```ignore
macro_rules! init_array {
    (@accum 0, $_e:expr) => {/* empty */};
    (@accum 1, $e:expr) => {$e};
    (@accum 2, $e:expr) => {$e, init_array!(@accum 1, $e)};
    (@accum 3, $e:expr) => {$e, init_array!(@accum 2, $e)};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            [init_array!(@accum $n, e)]
        }
    };
}
```

Ожидаем, что развертывание литералов из массива выполнится следующим образом:

```ignore
            [init_array!(@accum 3, e)]
            [e, init_array!(@accum 2, e)]
            [e, e, init_array!(@accum 1, e)]
            [e, e, e]
```

Однако, для это потребуется, чтобы каждый промежуточный шаг разворачивался в
незаконченное выражение. Даже при том, что промежуточные результаты никогда не
будут использоваться *снаружи* контекста макроса, это все равно запрещено.

Спихивание, в то же время, позволяет нам последовательно строить связку токенов,
не имея законченную структуру на каждом шаге. В примере на самом верху связка
вызовов макроса выполнится так:

```ignore
init_array! { String:: from ( "hi!" ) ; 3 }
init_array! { @ accum ( 3 , e . clone (  ) ) -> (  ) }
init_array! { @ accum ( 2 , e.clone() ) -> ( e.clone() , ) }
init_array! { @ accum ( 1 , e.clone() ) -> ( e.clone() , e.clone() , ) }
init_array! { @ accum ( 0 , e.clone() ) -> ( e.clone() , e.clone() , e.clone() , ) }
init_array! { @ as_expr [ e.clone() , e.clone() , e.clone() , ] }
```

Как видите, каждый слой добавляет накопление к выходу до тех пор пока
последнее правило не выделит все это в законченную структуру.

Единственным важным моментом в приведенной редакции является использование
`$($body:tt)*` для сохранения выхода без запуска разбора. `($input) ->
($output)` - это просто соглашение, призванное упростить понимание таких
макросов.

Спихиваемые накопления - часто используемая часть [последовательного потребителя TT](#incremental-tt-munchers), так как она позволяет конструировать промежуточные результаты произвольной сложности.
