# Haciéndolo Todo Genérico

Ya hemos tocado un poco los genéricos con Option y Box. Sin embargo, hasta
ahora hemos logrado evitar declarar cualquier tipo nuevo que sea realmente
genérico sobre elementos arbitrarios.

Resulta que en realidad es muy fácil. Hagamos todos nuestros tipos genéricos
ahora mismo:

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

Solo haces que todo sea un poco más puntiagudo, y de repente tu código es
genérico. Por supuesto, no podemos *simplemente* hacer esto, o el compilador
va a estar Super Enfadado.

```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

El problema está bastante claro: estamos hablando de esta cosa `List` pero eso
ya no es real. Como Option y Box, ahora siempre tenemos que hablar de
`List<Algo>`.

Pero ¿cuál es el Algo que usamos en todos estos impls? Al igual que List,
queremos que nuestras implementaciones funcionen con *todos* los T. Así que,
al igual que List, hagamos nuestros `impl`s puntiagudos:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...¡y eso es todo!


```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

Todo nuestro código es ahora completamente genérico sobre valores arbitrarios
de T. Caramba, Rust es *fácil*. Me gustaría hacer una mención especial a `new`
que ni siquiera cambió:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

Contempla la Gloria que es Self, guardián de la refactorización y la
programación copia-pega. También es interesante que no escribimos `List<T>`
cuando construimos una instancia de lista. Esa parte se infiere por nosotros
basándose en el hecho de que la estamos retornando desde una función que
espera un `List<T>`.

Muy bien, pasemos a *comportamiento* totalmente nuevo.