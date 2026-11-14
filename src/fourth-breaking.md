# Rompiendo (Breaking Down)

`pop_front` debería ser la misma lógica básica que `push_front`, pero al
revés. Intentemos:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // need to take the old head, ensuring it's -2
    self.head.take().map(|old_head| {                         // -1 old
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new
                // not emptying list
                new_head.borrow_mut().prev.take();            // -1 old
                self.head = Some(new_head);                   // +1 new
                // total: -2 old, +0 new
            }
            None => {
                // emptying list
                self.tail.take();                             // -1 old
                // total: -2 old, (no new)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

ACK. *RefCells*. Tengo que usar `borrow_mut` otravez, supongo...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*suspira*

> cannot move out of borrowed content

Hrm... Parece que Box nos estaba *realmente* mimando. `borrow_mut` solo nos da
un `&mut Node<T>`, ¡pero no podemos mover nada fuera de eso!

Necesitamos algo que tome un `RefCell<T>` y nos dé un `T`. Revisemos
[la documentación][refcell] para algo así:

> `fn into_inner(self) -> T`
>
> Consumes the RefCell, returning the wrapped value.

¡Eso parece prometedor!

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

Ah, caramba. `into_inner` quiere mover el RefCell fuera, pero no podemos,
porque está en un `Rc`. Como vimos en el capítulo anterior, `Rc<T>` solo
nos permite obtener referencias compartidas a sus internos. Eso tiene sentido,
porque ese es *el sentido mismo* de los punteros de contador de referencias:
¡son compartidos!

Esto fue un problema para nosotros cuando queríamos implementar Drop para
nuestra lista de contador de referencias, y la solución es la misma:
`Rc::try_unwrap`, que mueve los contenidos de un Rc si su contador de
referencias es 1.

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap` devuelve un `Result<T, Rc<T>>`. Los Results son básicamente
un `Option` generalizado, donde el caso `None` tiene datos asociados. En
este caso, el `Rc` que intentaste desenvolver (unwrap). Como no nos importa
el caso donde falla (si escribimos nuestro programa correctamente, *tiene* que
funcionar), simplemente llamamos a `unwrap` en él.

Como sea, veamos qué error del compilador obtenemos ahora (aceptémoslo, va a
haber uno).

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

UF. `unwrap` en Result requiere que puedas imprimir el caso de error con
debug. `RefCell<T>` solo implementa `Debug` si `T` lo hace. `Node` no
implementa Debug.

En lugar de hacer eso, evitemos el problema convirtiendo el Result a
un Option con `ok`:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

POR FAVOR.

```text
cargo build

```

SÍ.

*phew*

Lo logramos.

Implementamos `push` y `pop`.

Probemos robando el antiguo test básico de stack (porque es todo lo que
hemos implementado hasta ahora):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```

*Clavado*.

Ahora que podemos eliminar cosas de la lista adecuadamente, podemos
implementar Drop. Drop es un poco más interesante conceptualmente esta vez.
Mientras que antes nos molestamos en implementar Drop para nuestras pilas solo
para evitar la recursión sin límites, ahora necesitamos implementar Drop para
que *algo* ocurra en absoluto.

`Rc` no puede lidiar con ciclos. Si hay un ciclo, todo mantendrá todo lo demás
vivo. Una lista doblemente enlazada, resulta, es solo una gran cadena de
pequeños ciclos. Así que cuando soltamos nuestra lista, los dos nodos finales
tendrán sus contadores de referencia decrementados a 1... y entonces no
ocurrirá nada más. Bueno, si nuestra lista contiene exactamente un nodo,
estamos bien. Pero idealmente una lista debería funcionar correctamente si
contiene múltiples elementos. Quizás eso soy solo yo.

Como vimos, eliminar elementos fue un poco doloroso. Así que lo más fácil
para nosotros es simplemente hacer `pop` hasta obtener None:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(En realidad podríamos haber hecho esto con nuestras pilas mutables, ¡pero
los atajos son para personas que entienden las cosas!)

Podríamos considerar implementar las versiones `_back` de `push` y `pop`, pero
son solo trabajos de copiar y pegar que pospondremos para más adelante en el
capítulo. Por ahora, ¡veamos cosas más interesantes!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html