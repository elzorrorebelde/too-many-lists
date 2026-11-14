# Diseño (Layout)

Entonces, ¿cómo es una cola enlazada simple? Bueno, cuando teníamos
una pila enlazada simple, insertábamos (push) en un extremo de la
lista, y luego eliminábamos (pop) del mismo extremo. La única diferencia entre
una pila y una cola es que una cola elimina del *otro* extremo. Así que de
nuestra implementación de pila tenemos:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

stack push X:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

stack pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

Para hacer una cola, solo necesitamos decidir qué operación mover al
final de la lista: push, o pop? Como nuestra lista está enlazada
individualmente, podemos mover *cualquiera* de las operaciones al final con
el mismo esfuerzo.

Para mover `push` al final, simplemente caminamos hasta el `None` y lo
establecemos a Some con el nuevo elemento.

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

Para mover `pop` al final, simplemente caminamos hasta el nodo *anterior* al
None, y hacemos `take`:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

Podríamos hacer esto hoy y darlo por terminado, ¡pero eso apestaría! Ambas
operaciones recorren la lista *completa*. Algunos argumentarían que tal
implementación de cola es de hecho una cola porque expone la interfaz
correcta. Sin embargo, creo que las garantías de rendimiento son parte de la
interfaz. No me importan las cotas asintóticas precisas, solo "rápido" vs
"lento". Las colas garantizan que push y pop son rápidos, ¡y caminar sobre
toda la lista definitivamente no es rápido!

Los programadores puramente funcionales han ideado soluciones complejas para
colas con operaciones de tiempo constante en el peor caso, como la
[cola en tiempo real][real-time queue], usando técnicas como la pereza
(laziness), memoización y planificación. En lo que respecta a nuestro libro,
esto es hacer trampa. Reemplaza una buena comprensión de los punteros por
trucos matemáticos más o menos inteligentes. Lo ignoraremos felizmente y
mantendremos nuestra estructura de datos como una secuencia simple de nodos.

Una observación clave es que estamos desperdiciando un montón de trabajo
haciendo *lo mismo* una y otra vez. ¿Podemos "cachear" todo ese trabajo y
reutilizarlo? ¡Pues sí! ¡Podemos almacenar un puntero al final de la lista, y
simplemente saltar directamente allí!

Resulta que solo una inversión de `push` y `pop` funciona con esto.
Para invertir `pop` tendríamos que mover el puntero "tail" hacia atrás, pero
como nuestra lista está enlazada simple, no podemos hacer eso
eficientemente. Si en su lugar invertimos `push` solo tenemos que mover el
puntero "head" hacia adelante, lo cual es fácil.

Intentemos:

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NUEVO!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // Al insertar al final, next siempre es None
            next: None,
        });

        // Intercambia el tail antiguo por el nuevo tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // Si existía un tail antiguo, haz que apunte al nuevo tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // En caso contrario, haz que head apunte a él
                self.head = Some(new_tail);
            }
        }
    }
}
```

Como sea, veamos si esto compila:

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.31s
```

¡Excelente!

Ahora para `pop`, queremos eliminar de la cabeza. `pop` básicamente funciona
como nuestro pop de antes para la pila, excepto que ahora tenemos que asegurar
que `tail` también recibe mantenimiento:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        self.head = old_head.next;
        if self.head.is_none() {
            self.tail = None;
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fifth.rs:34:19
   |
34 |         self.head.take().map(|old_head| {
   |                        ^^^ cannot move out of borrowed content
   :
38 |             old_head.elem
   |             ^^^^^^^^^^^^^ cannot move out of borrowed content
```

Mmm, no está contento porque estamos usando `map` en un Option que estamos
obteniendo a través de un método de instancia. `map` toma el Option por valor,
y no debería hacerlo aquí. Podríamos evitarlo fácilmente separando el `take`
y el `map`:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    let old_head = self.head.take();
    old_head.map(|old_head| {
        self.head = old_head.next;
        if self.head.is_none() {
            self.tail = None;
        }
        old_head.elem
    })
}
```

```text
> cargo build

warning: unused import: `std::mem`
  --> src/fifth.rs:1:9
   |
1  | use std::mem;
   |         ^^^^^
   |
   = note: #[warn(unused_imports)] on by default
```

¡Un poco de una advertencia, pero compiló! Muy bien, podemos eliminar esa
importación de memoria. No la necesitábamos realmente porque estamos usando
el método `take` de Option, y manejando el None por nosotros mismos.



Bien, también deberíamos querer hacer flexibilizar tipos de las colas para
que también puedan usarse como pilas. Afortunadamente, como nuestra cola está
implementada sobre una pila, todo lo que tenemos que hacer es añadir los
métodos de la pila también. Ya hemos implementado la operación inversa pop,
así que necesitamos push inverso (que es el mismo que el push original de la
pila), o simplemente podemos exponer `push_front` y `pop_front` como
alias. Te dejaremos la decisión final a ti.

Ahora voy a avanzar un poco más rápido con los detalles de la implementación, ya que deberíamos estar cómodos con este tipo de código. No significa que debas esperar producirlo a la primera: estoy omitiendo parte del ensayo y error anterior. Cometí muchos errores al escribirlo, pero ver cómo olvido un `mut` o un `;` deja de ser instructivo después de cierto tiempo. ¡No te preocupes, veremos muchos otros mensajes de error!

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
```

¡Rayos! `Box` no implementa `Copy`, así que no podemos asignarlo a dos lugares. Más importante aún, `Box` *posee* aquello a lo que apunta y tratará de liberarlo cuando se destruya. Si `push` compilara así, liberaríamos dos veces el final de la lista. De hecho, tal como está escrito, liberaríamos el `old_tail` en cada `push`. ¡Qué desastre! 🙀

Sabemos cómo crear un puntero que no sea propietario: una referencia.

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NUEVO!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            next: None,
        });

        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

La idea es la misma que antes, pero aprovechamos el valor de retorno implícito
para extraer una referencia al `Node` del `Box` que acabamos de colocar.

```text
error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
```

Claro, las referencias dentro de los tipos necesitan tiempos de vida. ¿Cuál es
el tiempo de vida de esta referencia? Parece el mismo caso que `IterMut`, así
que añadamos un tiempo de vida genérico `'a`:

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NUEVO!
}

// ... el resto de la implementación es igual ...
```

El compilador nos da entonces un error de tiempos de vida: el préstamo de
`self` no dura lo suficiente para satisfacer `'a`. Probemos a declarar que
`self` se presta durante `'a`:

```rust ,ignore
pub fn push(&'a mut self, elem: T) {
```

¡Eso compila! Hagamos lo mismo con `pop`:

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;
        if self.head.is_none() {
            self.tail = None;
        }
        head.elem
    })
}
```

Pero una prueba sencilla revela el problema:

```text
error[E0499]: cannot borrow `list` as mutable more than once at a time
```

El compilador no se equivoca. Hemos cometido un pecado cardinal de Rust:
hemos almacenado una referencia a nosotros mismos *dentro de nosotros mismos*.
Cada llamada aislada a `push` o `pop` parece correcta, pero al usar la lista el
préstamo mutable de `self` dura hasta `'a` y bloquea todas las operaciones
posteriores.

La lista se ha tragado su propia cola y ha ascendido a un mundo de sueños.

> **NARRADOR:** Rust formalizó posteriormente la noción de un *pin* mediante
> [`Pin`][pin]. Es una herramienta importante para async/await y los futuros,
> donde una estructura puede contener referencias a sus propias variables
> locales. No queremos fijar nuestra lista, y este libro tampoco va a
> implementar un runtime asíncrono.

El problema también explica por qué guardar una referencia a nosotros mismos
puede ser peligroso:

```rust ,ignore
if self.head.is_none() {
    self.tail = None;
}
```

Si olvidáramos poner `tail` a `None`, apuntaría a un nodo que ya no pertenece a
la lista. Ese nodo se liberaría inmediatamente y tendríamos un puntero colgante.
Rust nos protege de ese peligro de una forma muy indirecta.

¿Volvemos al infierno de `Rc<RefCell>`? Por favor, no. En su lugar,
abandonaremos las referencias y usaremos punteros crudos. Nuestro diseño será:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // PELIGRO, PELIGRO
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

Y eso es todo. ¡Nada de esa débil combinación de conteo de referencias y
comprobación dinámica de préstamos! Punteros reales, duros y sin comprobar.

> **NARRADOR:** Esta implementación todavía era peligrosamente incorrecta,
> pero aún no había llegado el momento de aprender esa lección. La siguiente
> sección nos lo enseñará por las malas, como de costumbre.

Seamos C. Seamos C todo el día.

Estoy en casa. Estoy preparado.

Hola, `unsafe`.

> **NARRADOR:** Vaya, qué arrogancia tan increíble la del autor.

[real-time queue]: https://en.wikipedia.org/wiki/Queue_(abstract_data_type)#Real-time_queue
[pin]: https://doc.rust-lang.org/std/pin/index.html