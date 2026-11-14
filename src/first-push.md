# Push

Escribamos cómo insertar un valor en una lista. `push` *muta* la lista,
así que querremos usar `&mut self`. También necesitamos tomar un i32 para
insertar:

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        // TODO
    }
}
```

Lo primero es lo primero, necesitamos hacer un nodo para almacenar nuestro
elemento:

```rust ,ignore
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: ?????
        };
    }
```

¿Qué va en `next`? Bueno, ¡toda la lista anterior! ¿Podemos... simplemente
hacer eso?

```rust ,ignore
impl List {
    pub fn push(&mut self, elem: i32) {
        let new_node = Node {
            elem: elem,
            next: self.head,
        };
    }
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

Noooo. Rust nos está diciendo lo correcto, pero ciertamente no es obvio
qué significa exactamente, o qué hacer al respecto:

> cannot move out of borrowed content

Estamos intentando mover el campo `self.head` a `next`, pero Rust no quiere
que hagamos eso. Esto dejaría `self` solo parcialmente inicializado cuando
terminemos el préstamo (borrow) y "lo devolvamos" a su legítimo dueño. Como
dijimos antes, esa es la *única* cosa que no puedes hacer con un `&mut`:
Sería super grosero, y Rust es muy educado (también sería increíblemente
peligroso, pero seguramente *eso* no es por lo que le importa).

¿Qué pasa si devolvemos algo? A saber, el nodo que estamos creando:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: self.head,
    });

    self.head = Link::More(new_node);
}
```

```text
> cargo build
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:19:19
   |
19 |             next: self.head,
   |                   ^^^^^^^^^ cannot move out of borrowed content
```

No hay caso. En principio, esto es algo que Rust podría aceptar, pero no
lo hará (por varias razones — la más seria es [seguridad ante excepciones][exception safety]).
Necesitamos alguna forma de obtener la cabeza sin que Rust note que ha
desaparecido. Para pedir consejo, recurrimos al infame hacker de Rust Indiana
Jones:

![Indy se prepara para mem::replace](img/indy.gif)

Ah sí, Indy sugiere la maniobra `mem::replace`. Esta función increíblemente
útil nos permite robar un valor de un préstamo *reemplazándolo* con otro
valor. Importemos `std::mem` al principio del archivo, para que `mem` esté
en el ámbito local:

```rust ,ignore
use std::mem;
```

y usémoslo apropiadamente:

```rust ,ignore
pub fn push(&mut self, elem: i32) {
    let new_node = Box::new(Node {
        elem: elem,
        next: mem::replace(&mut self.head, Link::Empty),
    });

    self.head = Link::More(new_node);
}
```

Aquí `reemplazamos` self.head temporalmente con Link::Empty antes de
reemplazarlo con la nueva cabeza de la lista. No voy a mentir: esto es algo
bastante desafortunado que tener que hacer. Tristemente, debemos (por ahora).

Pero bueno, ¡`push` está terminado! Probablemente. Deberíamos probarlo,
honestamente. Ahora mismo la forma más fácil de hacerlo es probablemente
escribir `pop`, y asegurarnos de que produce los resultados correctos.





[exception safety]: https://doc.rust-lang.org/nightly/nomicon/exception-safety.html