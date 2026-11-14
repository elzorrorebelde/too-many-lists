# Pop

Al igual que `push`, `pop` quiere mutar la lista. A diferencia de `push`,
realmente queremos devolver algo. Pero `pop` también tiene que lidiar con un
caso límite complicado: ¿qué pasa si la lista está vacía? Para representar
este caso, usamos el confiable tipo `Option`:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>` es un enum que representa un valor que puede existir. Puede ser
`Some(T)` o `None`. Podríamos crear nuestro propio enum para esto como hicimos
con Link, pero queremos que nuestros usuarios puedan entender qué demonios es
nuestro tipo de retorno, y Option es tan ubicuo que *todo el mundo* lo sabe.
De hecho, es tan fundamental que se importa implícitamente en el ámbito de
cada archivo, así como sus variantes `Some` y `None` (para que no tengamos que
decir `Option::None`).

Los picos en `Option<T>` indican que Option es en realidad *genérico* sobre
T. ¡Eso significa que puedes hacer un Option para *cualquier* tipo!

Así que eh, tenemos este `Link`, ¿cómo averiguamos si está Empty o tiene
More? ¡Pattern matching con `match`!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

Ups, `pop` tiene que devolver un valor, y todavía no lo estamos haciendo.
Podríamos devolver `None`, pero en este caso probablemente es mejor devolver
`unimplemented!()`, para indicar que no hemos terminado de implementar la
función. `unimplemented!()` es una macro (`!` indica una macro) que entra en
pánico (panic) en el programa cuando llegamos a ella (~lo bloquea de forma
controlada).

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

Los pánicos incondicionales son un ejemplo de una [función divergente][diverging].
Las funciones divergentes nunca retornan a quien las llamó, por lo que pueden
usarse en lugares donde se espera un valor de cualquier tipo. Aquí,
`unimplemented!()` se está usando en lugar de un valor de tipo `Option<T>`.

Nótese también que no necesitamos escribir `return` en nuestro programa. La
última expresión (básicamente línea) en una función es implícitamente su
valor de retorno. Esto nos permite expresar cosas realmente simples de forma
más concisa. Siempre puedes retornar explícitamente con `return` como en
cualquier otro lenguaje tipo C.

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

¡Vamos Rust, déjanos en paz! Como siempre, Rust está super enfadado con
nosotros. Afortunadamente, esta vez también nos está dando la información
completa. Por defecto, un pattern match intentará mover sus contenidos a la
nueva rama, pero no podemos hacer esto porque no poseemos self por valor aquí.

```text
help: consider borrowing here: `&self.head`
```

Rust dice que deberíamos añadir una referencia a nuestro `match` para
arreglarlo. 🤷‍♀️ Probemos:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

Hurra, ¡compila otra vez! Ahora resolvamos esa lógica. Queremos hacer un
Option, así que creemos una variable para eso. En el caso Empty necesitamos
devolver None. En el caso More necesitamos devolver `Some(i32)`, y cambiar la
cabeza de la lista. Así que, ¿intentemos hacer básicamente eso?

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*cabeza*

*escritorio*

Estamos intentando mover algo de `node` cuando todo lo que tenemos es una
referencia compartida a él.

Deberíamos dar un paso atrás y pensar qué estamos intentando hacer. Queremos:

* Comprobar si la lista está vacía.
* Si está vacía, simplemente devolver None.
* Si *no* está vacía
    * eliminar la cabeza de la lista
    * eliminar su `elem`
    * reemplazar la cabeza de la lista con su `next`
    * devolver `Some(elem)`

La idea clave es que queremos *eliminar* cosas, lo que significa que queremos
obtener la cabeza de la lista *por valor*. Ciertamente no podemos hacerlo a
través de la referencia compartida que obtenemos con `&self.head`. También
"solo" tenemos una referencia mutable a `self`, así que la única forma de
mover cosas es *reemplazarlas*. ¡Parece que estamos haciendo el baile de Empty
otra vez!

Probemos:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

O M G

¡Compiló sin *ninguna* advertencia!

En realidad, voy a aplicar mi propia regla aquí: creamos este valor `result`
para devolverlo, pero en realidad no necesitábamos hacer eso en absoluto.
Así como una función se evalúa a su última expresión, cada bloque también se
evalúa a su última expresión. Normalmente suprimimos este comportamiento con
punto y coma, que en su lugar hace que el bloque se evalúe a la tupla vacía,
`()`. Este es en realidad el valor que devuelven las funciones que no declaran
un valor de retorno — como `push`.

Así que, en su lugar, podemos escribir `pop` como:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}
```

Que es un poco más conciso e idiomático. Nótese que la rama Link::Empty ha
perdido completamente sus llaves, porque solo tenemos una expresión que
evaluar. Solo una abreviatura útil para casos simples.

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

Bien, ¡sigue funcionando!


[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns