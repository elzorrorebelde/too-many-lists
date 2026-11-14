# Diseño (Layout)

Muy bien, de vuelta al tablero de dibujo con el diseño.

Lo más importante sobre una lista persistente es que puedes manipular las colas
de las listas básicamente gratis:

Por ejemplo, esta no es una carga de trabajo poco común con una lista
persistente:

```text
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

Pero al final queremos que la memoria se vea así:

```text
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

Esto simplemente no puede funcionar con Boxes, porque la propiedad de `B` es
*compartida*. ¿Quién debería liberarlo? Si suelto list2, ¿libera B? ¡Con boxes
ciertamente esperaríamos que sí!

Los lenguajes funcionales &mdash; y de hecho casi todos los demás lenguajes
&mdash; se salen con la suya usando *recolección de basura (garbage
collection)*. Con la magia de la recolección de basura, B se liberará solo
después de que todos dejen de mirarlo. ¡Hurra!

Rust no tiene nada como los recolectores de basura que tienen estos lenguajes.
Ellos tienen GC de *seguimiento (tracing)*, que excava a través de toda la
memoria que está presente en tiempo de ejecución y descubre qué es basura
automáticamente. En su lugar, todo lo que Rust tiene hoy es *conteo de
referencias (reference counting)*. El conteo de referencias puede considerarse
como un GC muy simple. Para muchas cargas de trabajo, tiene un rendimiento
significativamente menor que un recolector de seguimiento, y falla
completamente si consigues construir ciclos. ¡Pero bueno, es todo lo que
tenemos! Afortunadamente, para nuestro caso de uso nunca nos encontraremos con
ciclos (siéntete libre de intentar probar esto a ti mismo &mdash; yo seguro
que no lo haré).

Entonces, ¿cómo hacemos recolección de basura por conteo de referencias?
¡`Rc`! Rc es como Box, pero podemos duplicarlo, y su memoria *solo* se
liberará cuando *todos* los Rc derivados de él se suelten.
Desafortunadamente, esta flexibilidad tiene un costo serio: solo podemos tomar
una referencia compartida a sus internos. Esto significa que nunca podremos
realmente obtener datos de una de nuestras listas, ni mutarlas.

Entonces, ¿cómo va a ser nuestro diseño? Bueno, anteriormente teníamos:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

¿Podemos simplemente cambiar Box a Rc?

```rust ,ignore
// in third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```text
cargo build

error[E0412]: cannot find type `Rc` in this scope
 --> src/third.rs:5:23
  |
5 | type Link<T> = Option<Rc<Node<T>>>;
  |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
  |
1 | use std::rc::Rc;
  |
```

Oh vaya, qué quemada. A diferencia de todo lo que usamos para nuestras listas
mutables, Rc es tan poco importante que ni siquiera se importa implícitamente
en cada programa de Rust. *Qué perdedor*.

```rust ,ignore
use std::rc::Rc;
```

```text
cargo build

warning: field is never used: `head`
 --> src/third.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

Parece legítimo. Rust continúa siendo *completamente* trivial de escribir.
Apuesto a que podemos simplemente buscar y reemplazar Box por Rc y listo.

...

No. No podemos.