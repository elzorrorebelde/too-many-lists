# Diseño (Layout)

Empecemos estudiando la estructura de nuestro enemigo. Una lista doblemente enlazada es conceptualmente sencilla, pero ahí está precisamente su capacidad para engañarte y manipularte. Es el mismo tipo de lista enlazada que hemos visto una y otra vez, pero los enlaces van en ambas direcciones. El doble de enlaces, el doble de maldad.

Así que, en lugar de esto (eliminaremos `Some`/`None` para mantenerlo más limpio):

```text
... -> (A, ptr) -> (B, ptr) -> ...
```

Tenemos esto:

```text
... <-> (ptr, A, ptr) <-> (ptr, B, ptr) <-> ...
```

Esto permite recorrer la lista en cualquier dirección o desplazarse hacia delante y hacia atrás con un [cursor](https://doc.rust-lang.org/std/collections/struct.LinkedList.html#method.cursor_back_mut).

A cambio de esta flexibilidad, cada nodo debe almacenar el doble de punteros y cada operación tiene que ajustar muchos más punteros. Es una complicación considerable que facilita mucho cometer errores, así que haremos muchas pruebas.

Quizá también hayas notado que he decidido no dibujar los *extremos* de la lista. Esto se debe a que este es uno de los puntos en los que existen varias opciones defendibles para nuestra implementación. Sin duda necesitamos dos punteros: uno al principio de la lista y otro al final.

En mi opinión, hay dos formas destacables de hacerlo: el enfoque «tradicional» y el del «nodo ficticio».

El enfoque tradicional es una extensión sencilla de lo que hicimos con una pila: almacenar los punteros `head` y `tail` en la pila:

```text
[ptr, ptr] <-> (ptr, A, ptr) <-> (ptr, B, ptr)
  ^                                        ^
  +----------------------------------------+
```

Está bien, pero tiene una desventaja: los casos límite. Ahora hay dos extremos en nuestra lista, lo que significa el doble de casos límite. Es fácil olvidarse de uno y acabar con un error grave.

El enfoque del nodo ficticio intenta simplificar estos casos límite añadiendo a nuestra lista un nodo adicional que no contiene datos, pero conecta ambos extremos formando un anillo:

```text
[ptr] -> (ptr, ?DUMMY?, ptr) <-> (ptr, A, ptr) <-> (ptr, B, ptr)
           ^                                                 ^
           +-------------------------------------------------+ 
```

De este modo, cada nodo *siempre* tiene punteros reales a un nodo anterior y a uno siguiente de la lista. Incluso al eliminar el último elemento, basta con hacer que el nodo ficticio apunte hacia sí mismo:

```text
[ptr] -> (ptr, ?DUMMY?, ptr) 
           ^             ^
           +-------------+
```

Hay una parte de mí a la que esto le parece *muy* satisfactorio y elegante. Desafortunadamente, tiene un par de problemas prácticos:

Problema 1: una indirección y una asignación adicionales, especialmente para la lista vacía, que debe incluir el nodo ficticio. Algunas soluciones posibles son:

* No asignar el nodo ficticio hasta que se inserte algo: es sencillo y eficaz, pero vuelve a introducir algunos de los casos límite que intentábamos evitar usando punteros ficticios.

* Usar un nodo ficticio singleton vacío estático con copia al escribir, mediante algún esquema ingenioso que permita aprovechar las comprobaciones de copia al escribir junto con las comprobaciones normales. La verdad es que me tienta mucho, pero no podemos adentrarnos en ese camino oscuro en este libro. Lee el [código fuente de ThinVec](https://docs.rs/thin-vec/0.2.4/src/thin_vec/lib.rs.html#319-325) si quieres ver ese tipo de trucos perversos.

* Almacenar el nodo ficticio en la pila: no es práctico en un lenguaje sin constructores de movimiento al estilo de C++. Estoy seguro de que podríamos hacer algo extraño con el [anclaje](https://doc.rust-lang.org/std/pin/index.html), pero no lo haremos.

Problema 2: ¿qué *valor* se almacena en el nodo ficticio? Si es un entero no hay problema, pero ¿qué ocurre si almacenamos una lista llena de `Box`? ¡Podría ser imposible inicializar ese valor! Algunas soluciones posibles son:

* Hacer que cada nodo almacene `Option<T>`: sencillo y eficaz, pero también innecesariamente pesado y molesto.

* Hacer que cada nodo almacene [`MaybeUninit<T>`](https://doc.rust-lang.org/std/mem/union.MaybeUninit.html). Horrible y molesto.

* Un juego de tipos al estilo de la herencia, *muy* cuidadoso e ingenioso, para que el nodo ficticio no incluya el campo de datos. También resulta tentador, pero es extremadamente peligroso y molesto. Lee el [código fuente de `BTreeMap`](https://doc.rust-lang.org/1.55.0/src/alloc/collections/btree/node.rs.html#49-104) si quieres ver ese tipo de trucos perversos.

Para un lenguaje como Rust, los problemas superan con creces la comodidad, así que mantendremos el diseño tradicional. Usaremos el mismo diseño básico que empleamos para la cola insegura del capítulo anterior:

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

(Ahora que hemos llegado a la doble cola enlazada, por fin nos hemos ganado el derecho a llamarnos `LinkedList`, porque esta sí es la Verdadera Lista Enlazada.)

Este todavía no es del todo un diseño de calidad de producción *real*. Está *bien*, pero hay trucos mágicos que podemos usar para explicar mejor a Rust lo que estamos haciendo. Para ello tendremos que profundizar...