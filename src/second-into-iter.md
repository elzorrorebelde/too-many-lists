# IntoIter

Las colecciones se iteran en Rust usando el trait *Iterator*. Es un poco más
complicado que `Drop`:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Lo nuevo aquí es `type Item`. Esto declara que cada implementación de Iterator
tiene un *tipo asociado* llamado Item. En este caso, este es el tipo que puede
escupir cuando llamas a `next`.

La razón por la que Iterator produce `Option<Self::Item>` es porque la interfaz
unifica los conceptos de `has_next` y `get_next`. Cuando tienes el siguiente
valor, produces `Some(value)`, y cuando no, produces `None`. Esto hace que
la API sea generalmente más ergonómica y segura de usar e implementar,
evitando comprobaciones redundantes y lógica entre `has_next` y `get_next`.
¡Bien!

Tristemente, Rust no tiene nada como una declaración `yield` (aún), así que
vamos a tener que implementar la lógica nosotros mismos. También, hay en
realidad 3 tipos diferentes de iterador que cada colección debería esforzarse
por implementar:

* IntoIter — `T`
* IterMut — `&mut T`
* Iter — `&T`

En realidad ya tenemos todas las herramientas para implementar
IntoIter usando la interfaz de List: solo llamar a `pop` una y otra vez. Como
tal, implementaremos IntoIter como un envoltorio de nuevo tipo (newtype)
alrededor de List:

```rust ,ignore
// Las tuplas estructurales son una forma alternativa de struct,
// útil para envolver otros tipos de manera sencilla.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // Acceder numéricamente a los campos de una tupla estructural
        self.0.pop()
    }
}
```

Y escribamos una prueba:

```rust ,ignore
// --snip--
use super::Iterator;
// --snip--

#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

¡Bien!
