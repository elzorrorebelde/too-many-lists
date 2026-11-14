# Peek (Ojear)

Una cosa que ni siquiera nos molestamos en implementar la última vez fue el
peek (ojear). Hagámoslo. Todo lo que necesitamos hacer es devolver una
referencia al elemento en la cabeza de la lista, si existe. Suena fácil,
intentemos:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*Suspira*. ¿Y ahora qué, Rust?

Map toma `self` por valor, lo que movería el Option fuera de la cosa en la que
está. Anteriormente esto estaba bien porque lo acabábamos de `take`r, pero
ahora realmente queremos dejarlo donde estaba. La forma *correcta* de manejar
esto es con el método `as_ref` en Option, que tiene la siguiente definición:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

Degrada la `Option<T>` a un Option de una referencia a sus internos. Podríamos
hacer esto nosotros mismos con un match explícito pero *ugh no*. Significa
que necesitamos hacer una desreferencia extra para cortar a través de la
indirección adicional, pero afortunadamente el operador `.` maneja eso por
nosotros.


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

Clavado.

También podemos hacer una versión *mutable* de este método usando `as_mut`:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

EZ

No olvides probarlo:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Eso está bien, pero realmente no probamos si podíamos mutar ese valor de
retorno de `peek_mut`, ¿verdad? Si una referencia es mutable pero nadie la
muta, ¿realmente hemos probado la mutabilidad? Intentemos usar `map` en este
`Option<&mut T>` para poner un valor profundo:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

El compilador se queja de que `value` es inmutable, pero claramente escribimos
`&mut value`; ¿qué pasa? Resulta que escribir el argumento de la clausura de
esa manera no especifica que `value` sea una referencia mutable. En su lugar,
crea un patrón que se comparará con el argumento de la clausura;
`|&mut value|` significa "el argumento es una referencia mutable, pero solo
copia el valor al que apunta en `value`, por favor". Si solo usamos `|value|`,
el tipo de `value` será `&mut i32` y podemos realmente mutar la cabeza:

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

¡Mucho mejor!