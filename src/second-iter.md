# Iter

Muy bien, intentemos implementar Iter. Esta vez no podremos depender de que
List nos dé todas las características que queremos. Tendremos que crear el
nuestro propio. La lógica básica que queremos es mantener un puntero al nodo
actual que queremos producir el siguiente. Como ese nodo puede no existir (la
lista está vacía o hemos terminado de iterar), queremos que esa referencia sea
un Option. Cuando producimos un elemento, queremos avanzar al nodo `next` del
nodo actual.

Muy bien, intentemos eso:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

Oh, dios. Tiempos de vida (lifetimes). He oído hablar de estas cosas. He oído
que son una pesadilla.

Probemos algo nuevo: ¿ves esa cosa de `error[E0106]`? Es un código de error
del compilador. Podemos pedirle a rustc que los explique con, bueno,
`--explain`:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

Eso eh... en realidad no aclaró mucho (estos documentos asumen que entendemos
Rust mejor de lo que actualmente lo hacemos). Pero parece que deberíamos añadir
esas cosas `'a` a nuestro struct. Probemos eso.

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

Muy bien, estoy empezando a ver un patrón... añadamos estos pequeños a todo
lo que podamos:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

Oh, dios. Rompimos Rust.

Quizás deberíamos averiguar qué demonios significa esto de los tiempos de vida
`'a`.

Los tiempos de vida (lifetimes) pueden asustar a mucha gente porque son un
cambio a algo que hemos conocido y amado desde el amanecer de la
programación. En realidad hemos logrado esquivar los tiempos de vida hasta
ahora, aunque han estado enredados a lo largo de nuestros programas todo este
tiempo.

Los tiempos de vida son innecesarios en lenguajes con recolector de basura
porque el recolector de basura se asegura de que todo viva mágicamente tanto
como sea necesario. La mayoría de los datos en Rust se gestionan *manualmente*,
por lo que esos datos necesitan otra solución. C y C++ nos dan un claro ejemplo
de lo que pasa si simplemente dejas que la gente tome punteros a datos
aleatorios en la pila: una inseguridad generalizada e inmanejable. Esto se
puede dividir aproximadamente en dos clases de error:

* Tener un puntero a algo que salió del ámbito (scope)
* Tener un puntero a algo que fue mutado

Los tiempos de vida resuelven ambos problemas, y el 99% del tiempo, lo hacen
de una manera totalmente transparente.

Entonces, ¿qué es un tiempo de vida?

Muy simplemente, un tiempo de vida es el nombre de una región (~bloque/ámbito)
de código en algún lugar de un programa. Eso es todo. Cuando una referencia
está etiquetada con un tiempo de vida, estamos diciendo que tiene que ser
válida para esa región *completa*. Diferentes cosas ponen diferentes requisitos
sobre cuánto tiempo una referencia debe y puede ser válida. Todo el sistema de
tiempos de vida es a su vez solo un sistema de resolución de restricciones que
intenta minimizar la región de cada referencia. Si encuentra con éxito un
conjunto de tiempos de vida que satisface todas las restricciones, ¡tu programa
compila! Si no, obtienes un error diciendo que algo no vivió lo suficiente.

Dentro del cuerpo de una función generalmente no puedes hablar de tiempos de
vida, y tampoco querrías hacerlo *de todas formas*. El compilador tiene
información completa y puede inferir todas las restricciones para encontrar los
tiempos de vida mínimos. Sin embargo, a nivel de tipo y API, el compilador
*no* tiene toda la información. Requiere que le digas sobre la relación entre
diferentes tiempos de vida para que pueda averiguar qué estás haciendo.

En principio, esos tiempos de vida *también* podrían omitirse, pero entonces
verificar todos los préstamos sería un enorme análisis de programa completo que
produciría errores alucinantemente no locales. El sistema de Rust significa que
toda la verificación de préstamos se puede hacer en cada cuerpo de función de
forma independiente, y todos tus errores deberían ser bastante locales (o tus
tipos tienen firmas incorrectas).

¡Pero hemos escrito referencias en firmas de funciones antes, y estaba bien!
Eso es porque hay ciertos casos que son tan comunes que Rust elegirá
automáticamente los tiempos de vida por ti. Esto se llama *elisión de tiempos
de vida (lifetime elision)*.

En particular:

```rust ,ignore
// Solo una referencia en la entrada, así que la salida debe derivarse de esa entrada
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Muchas entradas, asumimos que son todas independientes
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Métodos, asumimos que todos los tiempos de vida de salida se derivan de `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

Entonces, ¿qué significa `fn foo<'a>(&'a A) -> &'a B` *en la práctica*? Todo
lo que significa es que la entrada debe vivir al menos tanto como la salida.
Así que si mantienes la salida durante mucho tiempo, esto expandirá la región
que la entrada debe ser válida. Una vez que dejas de usar la salida, el
compilador sabrá que está bien que la entrada también se vuelva inválida.

Con este sistema, Rust puede asegurar que nada se usa después de liberarse, y
que nada se muta mientras existen referencias pendientes. ¡Solo se asegura de
que todas las restricciones funcionen!

Muy bien. Entonces. Iter.

Retrocedamos al estado sin tiempos de vida:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

Necesitamos añadir tiempos de vida solo en las firmas de funciones y tipos:

```rust ,ignore
// Iter es genérico sobre *algún* tiempo de vida, no le importa
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// Sin tiempo de vida aquí, List no tiene ningún tiempo de vida asociado
impl<T> List<T> {
    // Declaramos un tiempo de vida nuevo aquí para el *exacto* préstamo
    // que crea el iter. Ahora &self necesita ser válido mientras el
    // Iter exista.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// *Sí* tenemos un tiempo de vida aquí, porque Iter tiene uno que necesitamos definir
impl<'a, T> Iterator for Iter<'a, T> {
    // Lo necesitamos aquí también, esto es una declaración de tipo
    type Item = &'a T;

    // Nada de esto necesita cambiar, manejado por lo anterior.
    // Self continúa siendo increíblemente genial y asombroso
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

Muy bien, creo que lo tenemos esta vez.

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

OK. ENTONCES. Arreglamos nuestros errores de tiempos de vida pero ahora
estamos obteniendo algunos errores de tipo nuevos.

Queremos estar almacenando `&Node`, pero estamos obteniendo `&Box<Node>`. Ok,
eso es bastante fácil, solo necesitamos desreferenciar el Box antes de tomar
nuestra referencia:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ﻿ ┻━┻

Olvidamos `as_ref`, así que estamos moviendo el box al `map`, lo que
significaría que se soltaría, y nuestras referencias colgarían:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref` añadió otra capa de indirección que necesitamos eliminar:


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
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

```text
cargo build

```

🎉 🎉 🎉

Las funciones as_deref y as_deref_mut son estables desde Rust 1.40. Antes de
eso necesitarías hacer `map(|node| &**node)` y `map(|node| &mut**node)`.
Puede que estés pensando "vaya, eso de `&**` es muy chapucero", y no estás
equivocado, pero como un buen vino, Rust mejora con el tiempo y ya no
necesitamos hacer tal cosa. Normalmente Rust es muy bueno haciendo este tipo de
conversión implícitamente, a través de un proceso llamado *deref coercion*,
donde básicamente puede insertar `*` a través de tu código para que
verifique tipos. ¡Puede hacerlo porque tenemos el verificador de préstamos
(borrow checker) para asegurarnos de que nunca estropeemos los punteros!

Pero en este caso la clausura en conjunción con el hecho de que tenemos un
`Option<&T>` en lugar de `&T` es un poco demasiado complicado para que
funcione, así que necesitamos ayudarlo siendo explícitos. Afortunadamente esto
es bastante raro, en mi experiencia.

Solo por completitud, *podríamos* darle una *pista diferente* con el
*turbofish*:

```rust ,ignore
    self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

Verás, map es una función genérica:

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
where
  F: FnOnce(T) -> U
```

El turbofish, `::<>`, nos permite decirle al compilador qué creemos que
deberían ser los tipos de esos genéricos. En este caso `::<&Node<T>, _>` dice
"debería devolver un `&Node<T>`, y no sé/no me importa ese otro tipo".

Esto a su vez permite al compilador saber que `&node` debería tener aplicada
la coerción de desreferencia (deref coercion), ¡así que no necesitamos aplicar
manualmente todos esos `*`!

Pero en este caso no creo que sea realmente una mejora, esto fue solo una
excusa mal disimulada para mostrar la coerción de desreferencia y el a veces
útil turbofish. 😅

Escribamos una prueba para asegurarnos de que no lo hicimos sin efecto o algo:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), None);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

¡Claro que sí!

También podemos notar que, a diferencia de `into_iter`, `iter` nos permite
iterar múltiples veces sobre la misma lista. Esta es la ventaja de usar `&self`
en lugar de `self` en la definición de la función, la función `iter` no
requirió que moviéramos la lista.

Finalmente, cabe notar que *podemos* aplicar la elisión de tiempos de vida
aquí:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}
```

es equivalente a:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```

¡Menos tiempos de vida, yupi!

O, si no te sientes cómodo "ocultando" que un struct contiene un tiempo de
vida, puedes usar la sintaxis de Rust 2018 de "tiempo de vida elidido
explícitamente", `'_`:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_deref() }
    }
}
```