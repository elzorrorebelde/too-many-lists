# Conceptos Básicos (Basics)

> **NARRADOR:** Esta sección contiene un error fundamental latente, porque ese es precisamente el objetivo del libro. Sin embargo, cuando empezamos a usar `unsafe`, es posible hacer las cosas mal y aun así conseguir que todo compile y *parezca* funcionar. El error fundamental se identificará en la siguiente sección. ¡No uses realmente el contenido de esta sección en código de producción!

Muy bien, volvamos a lo básico. ¿Cómo construimos nuestra lista?

Antes hacíamos lo siguiente:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

Pero ya no usamos `Option` para `tail`:

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found 
   |                                       enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

*Podríamos* usar un `Option`, pero, a diferencia de `Box`, `*mut` *puede* ser nulo. Esto significa que no puede beneficiarse de la optimización de puntero nulo. En su lugar, usaremos `null` para representar `None`.

Entonces, ¿cómo obtenemos un puntero nulo? Hay varias formas, pero prefiero usar
`std::ptr::null_mut()`. Si quieres, también puedes usar `0 as *mut _`, pero me
parece demasiado *sucio*.

```rust ,ignore
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

*shh*, compilador, pronto los usaremos.

Muy bien, volvamos a escribir `push`. Esta vez, en lugar de obtener un
`Option<&mut Node<T>>` después de insertar, obtendremos directamente un
`*mut Node<T>` que apunte al contenido del `Box`. Sabemos que esto es correcto
porque el contenido de un `Box` tiene una dirección estable, incluso si movemos
el `Box`. Por supuesto, esto no es *safe*, porque si simplemente eliminamos el
`Box`, tendremos un puntero a memoria liberada.

¿Cómo convertimos un puntero normal en un puntero crudo? ¡Coerciones! Si una
variable se declara como puntero crudo, una referencia normal se convertirá en él:

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

Ya tenemos toda la información necesaria. Podemos traducir nuestro código,
aproximadamente, a la versión anterior basada en referencias:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

¿Eh? Tenemos un puntero a un `Node`; ¿por qué no podemos acceder al campo `next`?

Rust se pone un poco pesado cuando usas punteros crudos. Para acceder al
contenido de un puntero crudo, exige que lo desreferenciemos manualmente, porque
es una operación insegura. Hagámoslo:

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             *self.tail.next = Some(new_tail);
   |             -----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

Uuuugh operator precedence.

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires 
              unsafe function or block

  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; 
     they can violate aliasing rules and cause data races: 
     all of these are undefined behavior
```

ESTO. NO. DEBERÍA. SER. TAN. DIFÍCIL.

¿Recuerdas que dije que Rust inseguro es como un lenguaje FFI para Rust seguro? El
compilador quiere que delimitemos explícitamente dónde hacemos ese trabajo de FFI.
Tenemos dos opciones. Primero, podemos marcar *toda* la función como `unsafe`,
en cuyo caso se convierte en una función de Rust inseguro y solo se puede llamar
dentro de un contexto `unsafe`. No es una buena opción, porque queremos que
nuestra lista sea segura de usar. Segundo, podemos escribir un bloque `unsafe`
dentro de la función para delimitar el límite de FFI. Esto declara que la
función completa es segura. Hagamos eso:


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    if !self.tail.is_null() {
        // Hello Compiler, I Know I Am Doing Something Dangerous And
        // I Promise To Be A Good Programmer Who Never Makes Mistakes.
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

Yay!

Es bastante interesante que ese sea el *único* lugar en el que hemos tenido que
escribir un bloque `unsafe` hasta ahora. Trabajamos con punteros crudos por todas
partes; ¿qué ocurre?

Resulta que Rust es un auténtico abogado puntilloso con las reglas cuando se
trata de `unsafe`. Y es razonable: queremos maximizar el conjunto de programas
de Rust seguro, porque son programas en los que podemos confiar mucho más. Para
conseguirlo, Rust delimita cuidadosamente una superficie mínima de inseguridad.
Fíjate en que, en todos los demás lugares donde hemos trabajado con punteros
crudos, solo los hemos *asignado* o hemos observado si eran nulos.

Si nunca desreferencias realmente un puntero crudo, *esas operaciones son
completamente seguras*. ¡Solo estás leyendo y escribiendo un entero! La única
forma de meterte en problemas con un puntero crudo es desreferenciarlo. Por eso
Rust considera insegura *únicamente* esa operación; todo lo demás es seguro.

Genial. Puntilloso. Pero técnicamente correcto.

> **NARRADOR:** En algún lugar al otro lado del mundo, una ingeniera de hardware
siente un escalofrío por la espalda: alguien debe de estar insistiendo otra vez
en que los punteros no son más que enteros. Mira su propuesta para un nuevo
sistema de autenticación de punteros por hardware y derrama una sola lágrima.
El ingeniero del compilador de al lado no siente nada: hace mucho que aprendió
a llevar siempre un jersey grueso.

Que solo algunas operaciones con punteros sean *realmente* inseguras plantea
un problema interesante: aunque se supone que debemos delimitar el alcance de
la inseguridad con el bloque `unsafe`, este depende de un estado establecido
fuera del bloque. ¡Incluso fuera de la función!

A esto lo llamo *contaminación* insegura. En cuanto usas `unsafe` en un módulo,
todo el módulo queda contaminado de inseguridad. Hay que escribirlo todo
correctamente para garantizar que se mantienen todas las invariantes del código
inseguro.

Esta contaminación es manejable gracias a la *privacidad*. Fuera de nuestro
módulo, todos los campos de nuestras estructuras son completamente privados,
así que nadie puede alterar nuestro estado de forma arbitraria. Mientras ninguna
combinación de las API expuestas provoque problemas, desde el punto de vista de
un observador externo todo nuestro código es seguro. Y, en realidad, esto no es
distinto del caso de FFI. A nadie le importa que una biblioteca matemática de
Python ejecute código C por debajo, siempre que exponga una interfaz segura.

En cualquier caso, pasemos a `pop`, que es prácticamente idéntico a la versión
basada en referencias:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

De nuevo vemos un caso en el que la seguridad depende del estado. Si no ponemos
a `null` el puntero `tail` en *esta* función, no veremos ningún problema. Sin
embargo, las llamadas posteriores a `push` empezarán a escribir en el `tail`
colgante.

Vamos a probarlo:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

Esta es simplemente la prueba de la pila, pero con los resultados esperados de
`pop` invertidos. También añadí algunos pasos al final para asegurarme de que no
se produzca el caso de corrupción del puntero `tail` en `pop`.

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

Gold Star!

> **NARRATOR:** Here it comes...