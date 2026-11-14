# Conceptos Básicos (Basics)

Ya conocemos muchos de los conceptos básicos de Rust, así que podemos hacer
muchas de las cosas simples otra vez.

Para el constructor, podemos nuevamente copiar y pegar:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

`push` y `pop` ya no tienen mucho sentido. En su lugar podemos proporcionar
`prepend` y `tail`, que proporcionan aproximadamente lo mismo.

Empecemos con prepend (anteponer). Toma una lista y un elemento, y devuelve
una Lista. Como en el caso de la lista mutable, queremos hacer un nuevo nodo,
que tenga la lista anterior como su valor `next`. Lo único novedoso es cómo
*obtener* ese valor next, porque no se nos permite mutar nada.

La respuesta a nuestras oraciones es el trait Clone. Clone es implementado por
casi todos los tipos, y proporciona una forma genérica de obtener "otro como
este" que sea lógicamente disjunto, dado solo una referencia compartida. Es
como un constructor de copia en C++, pero nunca se invoca implícitamente.

Rc en particular usa Clone como la forma de incrementar el contador de
referencias. Así que en lugar de mover un Box para estar en la sublista,
simplemente clonamos la cabeza de la lista anterior. Ni siquiera necesitamos
hacer match en la cabeza, porque Option expone una implementación de Clone que
hace exactamente lo que queremos.

Muy bien, intentémoslo:

```rust ,ignore
pub fn prepend(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

Vaya, Rust es realmente estricto en cuanto a usar realmente los campos. ¡Puede
decir que ningún consumidor puede realmente observar el uso de estos campos!
Aún así, parece que vamos bien hasta ahora.

`tail` es la inversa lógica de esta operación. Toma una lista y devuelve toda
la lista con el primer elemento eliminado. Todo lo que es es clonar el
*segundo* elemento en la lista (si existe). Probemos:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

Hrm, nos equivocamos. `map` espera que devolvamos una Y, pero aquí estamos
devolviendo un `Option<Y>`. Afortunadamente, este es otro patrón común de
Option, y podemos usar `and_then` para permitirnos devolver un Option.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

Genial.

Ahora que tenemos `tail`, probablemente deberíamos proporcionar `head`, que
devuelve una referencia al primer elemento. Eso es simplemente `peek` de la
lista mutable:

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

```text
> cargo build

```

Bien.

Esa es suficiente funcionalidad para probarlo:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.prepend(1).prepend(2).prepend(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

¡Perfecto!

Iter también es idéntico a como era para nuestra lista mutable:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().prepend(1).prepend(2).prepend(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), None);
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

¿Quién dijo que la tipificación dinámica era más fácil?

(los perdedores)

Nótese que no podemos implementar IntoIter o IterMut para este tipo. Solo
tenemos acceso compartido a los elementos.