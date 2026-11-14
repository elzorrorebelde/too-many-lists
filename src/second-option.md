# Usando Option

Los lectores particularmente observadores pueden haber notado que en realidad
reinventamos una versión realmente mala de Option:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link es simplemente `Option<Box<Node>>`. Ahora, está bien no tener que escribir
`Option<Box<Node>>` en todas partes, y a diferencia de `pop`, no estamos
exponiendo esto al mundo exterior, así que quizás está bien. Sin embargo,
Option tiene algunos métodos *realmente buenos* que hemos estado implementando
manualmente nosotros mismos. No hagamos *eso*, y reemplacémoslo todo con
Options. Primero, lo haremos ingenuamente simplemente renombrando todo para
usar Some y None:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

Esto es marginalmente mejor, pero las grandes victorias vendrán de los métodos
de Option.

Primero, `mem::replace(&mut option, None)` es un modismo tan
increíblemente común que Option simplemente lo convirtió en un método: `take`.

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

Segundo, `match option { None => None, Some(x) => Some(y) }` es un modismo
tan increíblemente común que se llamó `map`. `map` toma una función para
ejecutar sobre la `x` en el `Some(x)` para producir la `y` en `Some(y)`.
Podríamos escribir una `fn` apropiada y pasarla a `map`, pero preferiríamos
escribir qué hacer *en línea (inline)*.

La forma de hacer esto es con una *clausura (closure)*. Las clausuras son
funciones anónimas con un superpoder extra: ¡pueden referirse a variables
locales *fuera* de la clausura! Esto las hace super útiles para todo tipo
de lógica condicional. El único lugar donde hacemos un `match` es en `pop`,
así que reescribamos eso:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

Ah, mucho mejor. Asegurémonos de que no rompimos nada:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

¡Genial! Pasemos a mejorar realmente el *comportamiento* del código.