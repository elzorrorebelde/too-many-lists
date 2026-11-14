# Iteración

Intentemos iterar este chico malo.

## IntoIter

IntoIter, como siempre, va a ser el más fácil. Solo envuelve la pila y
llama a `pop`:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop_front()
    }
}
```

Pero tenemos un desarrollo interesante nuevo. Donde antes solo había
un orden de iteración "natural" para nuestras listas, un Deque es
inherentemente bidireccional. ¿Qué tiene de especial el orden de adelante
hacia atrás? ¿Qué pasa si alguien quiere iterar en la otra dirección?

Rust realmente tiene una respuesta para esto: `DoubleEndedIterator`.
DoubleEndedIterator *hereda* de Iterator (lo que significa que todos los
DoubleEndedIterator son Iterators) y requiere un nuevo método: `next_back`.
Tiene la misma firma exacta que `next`, pero se supone que debe producir
elementos desde el otro extremo. La semántica de DoubleEndedIterator es
super conveniente para nosotros: el iterador se convierte en un deque.
Puedes consumir elementos desde el frente y el fondo hasta que los dos
extremos converjan, momento en el cual el iterador está vacío.

Al igual que Iterator y `next`, resulta que `next_back` no es realmente
diferente de simplemente llamar a `pop_back`. Pero necesitamos ponerlo
detrás de un trait para que la maquinaria de `for` loops, `map`, `filter`,
etc., puedan usarlo.

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

```text
> cargo build
```

Awww yeah.

Probémoslo:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::iter ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

¡Sí!

## Iter e IterMut

¿Podríamos implementar Iter para nuestro deque seguro?

Bueno, necesitamos devolver una referencia inmutable a los elementos de la
lista, pero como usamos RefCells internamente, los usuarios necesitan usar Refs
para obtener inmutabilidad.

En realidad no podemos ocultarlo. Simplemente no podemos.

Aquí está el truco: hemos creado una abstracción que se fuga. Los
consumidores de nuestra API necesitan saber sobre Refs, RefCells y RefMut
para iterar sobre nuestra deque segura.

Aceptémoslo. No implementemos Iter e IterMut. Es demasiado difícil y
desordenado.

Pasemos a la siguiente lista para sentirnos mejor con nosotros mismos.