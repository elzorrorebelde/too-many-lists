# Ojeando (Peeking)

Muy bien, sobrevivimos a `push` y `pop`. No voy a mentir, se puso un poco
emocional allí. La corrección en tiempo de compilación es una droga poderosa.

Relajémonos haciendo algo simple: implementemos `peek_front`. Eso siempre fue
muy fácil antes. Tiene que seguir siendo fácil, ¿verdad?

¿Verdad?

De hecho, ¡creo que puedo copiarlo y pegarlo!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

Espera. Esta vez no.

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // PRÉSTAMO (BORROW)!!!!
        &node.borrow().elem
    })
}
```

JA.

```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

Ok, solo estoy quemando mi computadora.

Esta es exactamente la misma lógica que nuestra pila enlazada simple.
¿Por qué son diferentes las cosas? POR QUÉ.

La respuesta es realmente toda la moraleja de este capítulo: RefCells hacen
que todo sea tristeza. Hasta ahora, RefCells solo habían sido una molestia.
Ahora se van a convertir en una pesadilla.

Entonces, ¿qué está pasando? Para entenderlo, necesitamos volver a la
definición de `borrow`:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

En la sección de diseño dijimos:

> En lugar de aplicarlo estáticamente, RefCell las aplica en tiempo de
> ejecución. Si rompes las reglas, RefCell simplemente entrará en pánico y
> bloqueará el programa. ¿Por qué devuelve estas cosas Ref y RefMut? Bueno,
> básicamente se comportan como `Rc`s pero para préstamos (borrowing). También
> mantienen el RefCell prestado hasta que salen del ámbito.
> ***Llegaremos a eso más tarde.***

Es más tarde.

`Ref` y `RefMut` implementan `Deref` y `DerefMut` respectivamente. Así que
para la mayoría de los propósitos se comportan *exactamente* como `&T` y
`&mut T`. Sin embargo, debido a cómo funcionan esos traits, la referencia que
se devuelve está conectada al tiempo de vida del Ref, y no al RefCell real.
Esto significa que el Ref tiene que estar presente mientras mantengamos la
referencia.

Esto es de hecho necesario para la corrección. Cuando un Ref se suelta, le
dice al RefCell que ya no está prestado. Así que si *lográramos* mantener
nuestra referencia por más tiempo del que existió el Ref, podríamos obtener
un RefMut mientras una referencia estuviera presente y romper totalmente el
sistema de tipos de Rust por la mitad.

Entonces, ¿dónde nos deja eso? Solo queremos devolver una referencia, pero
necesitamos mantener esta cosa Ref cerca. Pero tan pronto como devolvemos la
referencia desde `peek`, la función termina y el `Ref` sale del ámbito.

😖

Hasta donde sé, en realidad estamos totalmente muertos aquí. No puedes
encapsular totalmente el uso de RefCells así.

Pero... ¿qué pasa si simplemente renunciamos a ocultar totalmente nuestros
detalles de implementación? ¿Qué pasa si devolvemos Refs?

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

Blurp. Tengo que importar algunas cosas.


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

Hmm... es cierto. Tenemos un `Ref<Node<T>>`, pero queremos un `Ref<T>`.
Podríamos abandonar toda esperanza de encapsulación y simplemente devolver
eso. También podríamos hacer las cosas aún más complicadas y envolver
`Ref<Node<T>>` en un nuevo tipo para exponer solo acceso a un `&T`.

Ambas opciones son *algo* malas.

En su lugar, vamos a ir más profundo. Divirtámonos un poco. Nuestra fuente
de diversión es *esta bestia*:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> Make a new Ref for a component of the borrowed data.

Sí: así como puedes mapear sobre un Option, puedes mapear sobre un Ref.

Estoy seguro de que en algún lugar alguien está realmente emocionado por
*mónadas* o lo que sea, pero no me importa nada de eso. Tampoco creo que sea
una mónada propiamente dicha ya que no hay un caso tipo None, pero divago.

Es genial y eso es todo lo que me importa. *Necesito esto*.

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

Awww yissss

Asegurémonos de que esto funciona adaptando el test de nuestra pila.
Necesitamos hacer algunos ajustes para lidiar con el hecho de que los Refs
no implementan comparaciones.

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

¡Genial!