# Send, Sync y Pruebas de Compilación

En realidad nos queda otro par de traits en los que pensar, pero son especiales. Tenemos que tratar con el Sacro Imperio Romano de Rust: los traits integrados inseguros de activación explícita (OIBIT): [Send y Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html), que en realidad se desactivan por defecto y se implementan automáticamente (¡1 de 3 no está mal!).

Al igual que `Copy`, estos traits no tienen código asociado; son simplemente marcadores que indican que un tipo posee una propiedad concreta. `Send` indica que es seguro enviar el tipo a otro hilo. `Sync` indica que es seguro compartirlo entre hilos (`&Self: Send`).

Aquí se aplica el mismo argumento de que `LinkedList` sea covariante: en general, las colecciones normales que no usan trucos sofisticados de mutabilidad interior pueden implementar `Send` y `Sync` de forma segura.

Pero he dicho que se desactivan por defecto. Entonces, ¿ya lo están en nuestro caso? ¿Cómo podríamos saberlo?

Añadamos algo de magia al código: basura privada que no compile a menos que nuestros tipos tengan las propiedades esperadas:

```rust ,ignore
#[allow(dead_code)]
fn assert_properties() {
    fn is_send<T: Send>() {}
    fn is_sync<T: Sync>() {}

    is_send::<LinkedList<i32>>();
    is_sync::<LinkedList<i32>>();

    is_send::<IntoIter<i32>>();
    is_sync::<IntoIter<i32>>();

    is_send::<Iter<i32>>();
    is_sync::<Iter<i32>>();

    is_send::<IterMut<i32>>();
    is_sync::<IterMut<i32>>();

    is_send::<Cursor<i32>>();
    is_sync::<Cursor<i32>>();

    fn linked_list_covariant<'a, T>(x: LinkedList<&'static T>) -> LinkedList<&'a T> { x }
    fn iter_covariant<'i, 'a, T>(x: Iter<'i, &'static T>) -> Iter<'i, &'a T> { x }
    fn into_iter_covariant<'a, T>(x: IntoIter<&'static T>) -> IntoIter<&'a T> { x }
}
```

```text
cargo build
   Compiling linked-list v0.0.3 
error[E0277]: `NonNull<Node<i32>>` cannot be sent between threads safely
   --> src\lib.rs:433:5
    |
433 |     is_send::<LinkedList<i32>>();
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ `NonNull<Node<i32>>` cannot be sent between threads safely
    |
    = help: within `LinkedList<i32>`, the trait `Send` is not implemented for `NonNull<Node<i32>>`
    = note: required because it appears within the type `Option<NonNull<Node<i32>>>`
note: required because it appears within the type `LinkedList<i32>`
   --> src\lib.rs:8:12
    |
8   | pub struct LinkedList<T> {
    |            ^^^^^^^^^^
note: required by a bound in `is_send`
   --> src\lib.rs:430:19
    |
430 |     fn is_send<T: Send>() {}
    |                   ^^^^ required by this bound in `is_send`

<a million more errors>
```

¡Pero qué demonios! ¡Tenía un chiste buenísimo sobre el Sacro Imperio Romano!

Te mentí cuando dije que los punteros crudos solo tenían una protección de seguridad: esta es la otra. `*const` y `*mut` se excluyen explícitamente de `Send` y `Sync` para estar a salvo, así que *realmente* tenemos que volver a incluirlos:

```rust ,ignore
unsafe impl<T: Send> Send for LinkedList<T> {}
unsafe impl<T: Sync> Sync for LinkedList<T> {}

unsafe impl<'a, T: Send> Send for Iter<'a, T> {}
unsafe impl<'a, T: Sync> Sync for Iter<'a, T> {}

unsafe impl<'a, T: Send> Send for IterMut<'a, T> {}
unsafe impl<'a, T: Sync> Sync for IterMut<'a, T> {}
```

Ten en cuenta que aquí tenemos que escribir una *implementación `unsafe`*: ¡son traits *unsafe*! El código inseguro, como las bibliotecas de concurrencia, puede confiar en nosotros únicamente si implementamos estos traits correctamente. Como no hay código real, la garantía que ofrecemos es simplemente que sí, es seguro enviar o compartir estos tipos entre hilos.

No añadas estas implementaciones a la ligera, pero como profesional certificado puedo decir que aquí son totalmente correctas. Observa que no necesitamos implementar `Send` y `Sync` para `IntoIter`: solo contiene una `LinkedList`, así que obtiene `Send` y `Sync` automáticamente &mdash; Te dije que en realidad se excluían por defecto. (La sintaxis para excluirse es la hilarante of `impl !Send for MyType {}`.)

```text
cargo build
   Compiling linked-list v0.0.3
    Finished dev [unoptimized + debuginfo] target(s) in 0.18s
```

¡Perfecto!

...Espera, en realidad sería muy peligroso que algo que *no debería* tener estas propiedades las tuviera. En particular, `IterMut` *definitivamente* no debería ser covariante, porque es «como» `&mut T`. Pero ¿cómo podemos comprobarlo?

¡Con magia! Bueno, en realidad con `rustdoc`. No tenemos que usar `rustdoc` para esto, pero es la forma más divertida. Si escribes un comentario de documentación e incluyes un bloque de código, `rustdoc` intentará compilarlo y ejecutarlo, así que podemos usarlo para crear «programas» anónimos que no afecten al programa principal:


```rust ,ignore
    /// ```
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
error[E0308]: mismatched types
 --> src\lib.rs:461:86
  |
6 | fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
  |                                                                                      ^ lifetime mismatch
  |
  = note: expected struct `linked_list::IterMut<'_, &'a T>`
             found struct `linked_list::IterMut<'_, &'static T>`
```

De acuerdo, hemos demostrado que es invariante, pero ahora fallan las pruebas. No pasa nada: `rustdoc` permite indicar que el fallo es esperado anotando el bloque con `compile_fail`.

(En realidad solo hemos demostrado que «no es covariante», pero, sinceramente, si consigues hacer que un tipo sea «contravariante de forma accidental e incorrecta», enhorabuena.)

```rust ,ignore
    /// ```compile_fail
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

```text
cargo test
   Compiling linked-list v0.0.3
    Finished test [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src\lib.rs

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

¡Bien! Recomiendo escribir siempre primero la prueba sin `compile_fail`, para confirmar que no compila *por el motivo correcto*. Por ejemplo, esa prueba también fallaría (y por tanto pasaría) si olvidaras el `use`, y eso no es lo que queremos. Aunque conceptualmente resulte atractivo poder «exigir» un error concreto al compilador, sería una pesadilla absoluta que convertiría en un cambio incompatible el hecho de que el compilador produjera mejores errores. Queremos que el compilador mejore, así que no podemos exigir eso.

(Espera, en realidad podemos especificar el código de error que queremos junto a `compile_fail`, **pero esto solo funciona en `nightly` y es mala idea depender de ello por los motivos anteriores. En versiones que no sean `nightly` se ignorará silenciosamente.**)

```rust ,ignore
    /// ```compile_fail,E0308
    /// use linked_list::IterMut;
    /// 
    /// fn iter_mut_covariant<'i, 'a, T>(x: IterMut<'i, &'static T>) -> IterMut<'i, &'a T> { x }
    /// ```
    fn iter_mut_invariant() {}
```

...además, ¿te fijaste en que realmente hicimos que `IterMut` fuera invariante? Era fácil pasarlo por alto, porque simplemente copié y pegué `Iter` y lo añadí al final. Es la última línea de este bloque:

```rust ,ignore
pub struct IterMut<'a, T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<&'a mut T>,
}
```

Probemos a eliminar ese `PhantomData`:

```text
 cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
error[E0392]: parameter `'a` is never used
  --> src\lib.rs:30:20
   |
30 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, or using a marker such as `PhantomData`
```

¡Ja! El compilador nos protege y no permite simplemente que *no* usemos el tiempo de vida. Probemos en su lugar el ejemplo *incorrecto*:

```rust ,ignore
    _boo: PhantomData<&'a T>,
```

```text
cargo build
   Compiling linked-list v0.0.3 (C:\Users\ninte\dev\contain\linked-list)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
```

¡Compila! ¿Detectan ahora algún problema nuestras pruebas?

```text
cargo test

...

   Doc-tests linked-list

running 1 test
test src\lib.rs - assert_properties::iter_mut_invariant (line 458) - compile fail ... FAILED

failures:

---- src\lib.rs - assert_properties::iter_mut_invariant (line 458) stdout ----
Test compiled successfully, but it's marked `compile_fail`.

failures:
    src\lib.rs - assert_properties::iter_mut_invariant (line 458)

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s
```

¡¡¡Sí!!! ¡El sistema funciona! Me encanta tener pruebas que realmente cumplen su función, para no tener que horrorizarme tanto ante errores inminentes.

