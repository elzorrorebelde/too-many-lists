# Diseño (Layout)

Entonces, ¿cómo es una cola enlazada individualmente? Bueno, cuando teníamos
una pila enlazada individualmente, insertábamos (push) en un extremo de la
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
como nuestra lista está enlazada individualmente, no podemos hacer eso
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

;
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            next: None,
        });

        // Replace the list's old tail with the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        // Now we need to fix the old tail's next pointer
        match old_tail {
            // If there was an old tail, point it at the new guy
            Some(mut old_tail) => old_tail.next = Some(new_tail),
            // Otherwise the list was empty, set the head too
            None => self.head = Some(new_tail),
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

[real-time queue]: https://en.wikipedia.org/wiki/Queue_(abstract_data_type)#Real-time_queue