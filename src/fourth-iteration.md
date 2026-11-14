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

¿Podríamos implementar `Iter` para nuestra doble cola segura?

Será bastante menos sencillo. Tenemos que devolver referencias inmutables a
los elementos de la lista, pero internamente usamos `RefCell`, así que los
usuarios tendrían que trabajar con `Ref` para obtener esa inmutabilidad.

En realidad, no podemos ocultar ese detalle. Hemos creado una abstracción que
se filtra: los consumidores de nuestra API tendrían que conocer `Ref`,
`RefCell` y `RefMut` para iterar sobre nuestra doble cola segura.

Podríamos intentar almacenar un `Ref` dentro del iterador:

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

A primera vista parece razonable. El problema aparece al implementar `next`:
para devolver el elemento actual y conservar el siguiente nodo necesitamos
separar un `Ref` en dos préstamos distintos. `Ref::map` solo puede transformar
el préstamo completo, y el siguiente préstamo queda ligado al `Ref` que
intentamos consumir:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;

    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

La función que parece adecuada existe y se llama [`map_split`][map-split]:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

Pero tampoco resuelve el problema. Cada vez que avanzamos por la lista nos
adentramos en otro `RefCell`, así que acabaríamos necesitando conservar una
pila de `Ref` para representar todos los préstamos pendientes. Cuando dejamos
de mirar un elemento, tendríamos que reducir el contador de préstamos de todos
los `RefCell` anteriores.

Podríamos intentar salir de los `RefCell` usando los `Rc`. Si el iterador
clonara el `Rc` completo, obtendríamos un identificador propietario hacia el
interior de la lista:

```rust ,ignore
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}
```

Pero ¿qué devolveríamos entonces? ¿`&T`? ¿`Ref<T>`? Ninguna de las dos
opciones funciona: el iterador ya no tiene un tiempo de vida y ambas referencias
necesitan uno declarado de antemano. Podríamos buscar una forma de transformar
el `Rc<Node<T>>` en un `Rc<T>` mediante una abstracción como
[`owning_ref`][own-ref], pero eso introduce otro problema: la invalidación del
iterador.

Hasta ahora éramos inmunes a la invalidación porque `Iter` tomaba prestada la
lista y la mantenía inmutable. Si el iterador devolviera `Rc`, no tomaría
prestada la lista, de modo que el usuario podría llamar a `push` y `pop` al
mismo tiempo que conserva punteros hacia ella. Insertar elementos sería
relativamente inocuo, pero eliminar el nodo al que apunta el iterador acabaría
provocando un fallo al intentar recuperar el `Rc` con `try_unwrap`.

Eso es interesante: podemos tener punteros propietarios hacia el interior de
la lista y modificarla a la vez, y todo funciona hasta que intentamos eliminar
un nodo observado por el iterador. Incluso entonces no obtenemos punteros
colgantes: el programa falla de forma determinista. Sin embargo, tener que
resolver la invalidación además de mapear `Rc` parece una mala solución.

`Rc<RefCell>` nos ha fallado finalmente. La mutabilidad interior es estupenda
para escribir *aplicaciones* seguras, pero no tanto para escribir *bibliotecas*
seguras con una API limpia. Podríamos hacer que funcionara si expusiéramos
`Node` por todas partes, e incluso podríamos crear varios `IterMut` cuya
exclusión mutua se comprobara en tiempo de ejecución, pero eso sería una API
terrible.

Así que abandonamos `Iter` e `IterMut` en esta implementación. Se podrían
hacer, pero el coste en complejidad y ergonomía no merece la pena. Pasemos a la
siguiente lista para sentirnos un poco mejor.

[own-ref]: https://crates.io/crates/owning_ref
[map-split]: https://doc.rust-lang.org/std/cell/struct.Ref.html#method.map_split