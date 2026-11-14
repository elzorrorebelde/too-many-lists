# Diseño (Layout)

La clave de nuestro diseño es el tipo `RefCell`. El corazón de
RefCell es un par de métodos:

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

Las reglas para `borrow` y `borrow_mut` son exactamente las de `&` y `&mut`:
puedes llamar a `borrow` tantas veces como quieras, pero `borrow_mut` requiere
exclusividad.

En lugar de aplicarlo estáticamente, RefCell las aplica en tiempo de
ejecución. Si rompes las reglas, RefCell simplemente entrará en pánico y
bloqueará el programa. ¿Por qué devuelve estas cosas Ref y RefMut? Bueno,
básicamente se comportan como `Rc`s pero para préstamos (borrowing). También
mantienen el RefCell prestado hasta que salen del ámbito. Llegaremos a eso
más tarde.

Ahora con Rc y RefCell podemos convertirnos en... un lenguaje increíblemente
verboso, pervasiveamente mutable, con recolector de basura, que no puede
recolectar ciclos. ¡V-viva...

Muy bien, queremos ser *doblemente enlazados*. Esto significa que cada nodo
tiene un puntero al nodo anterior y al siguiente. Además, la lista misma tiene
un puntero al primer y último nodo. Esto nos da inserción y eliminación rápida
en *ambos* extremos de la lista.

Así que probablemente queremos algo como:

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

Oye, ¡compiló! ¡Muchas advertencias de código muerto, pero compiló!
Intentemos usarlo.