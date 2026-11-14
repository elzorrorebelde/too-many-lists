# Diseño y conceptos básicos 2: punteros crudos

> Resumen de las tres secciones anteriores: mezclar aleatoriamente punteros seguros como `&`, `&mut` y `Box` con punteros inseguros como `*mut` y `*const` es una receta para provocar comportamiento indefinido, porque los punteros seguros introducen restricciones adicionales que no respetamos con los punteros crudos.

Dios mío, tengo que volver a escribir listas enlazadas. Vale. ¡VALE! Todo irá bien.

Vamos a avanzar rápidamente por gran parte de esta sección, ya que ya hablamos del diseño en el primer intento y todo lo que hicimos era básicamente correcto, salvo por la forma en que mezclamos punteros seguros e inseguros.


# Layout

En el nuevo diseño solo usaremos punteros crudos, todo será perfecto y nunca volveremos a cometer errores.

Este es nuestro diseño antiguo y defectuoso:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // INNOCENT AND KIND
}

type Link<T> = Option<Box<Node<T>>>; // THE REAL EVIL

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

Y este es nuestro nuevo diseño:

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // MUCH BETTER

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

Recuerda: `Option` no es tan práctico ni útil cuando usamos punteros crudos, así que ya no lo utilizaremos. En secciones posteriores veremos el tipo `NonNull`, pero no te preocupes por eso todavía.



# Basics

`List::new` es prácticamente igual.

```rust ,ignore
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

`push` es básicamente lo m-


```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(
```

Espera, ya no usamos `Box`. ¿Cómo asignamos memoria sin `Box`?

Bueno, *podríamos* hacerlo con `std::alloc::alloc`, pero sería como llevar una katana a la cocina. Cumpliría su cometido, pero sería un exceso y resultaría poco manejable.

We want to *have* boxes, but, *not*. One completely wild but *maybe* viable option would be to do something like this:

```rust ,ignore
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

La idea sería crear los `Box` y almacenarlos en nuestro nodo, pero después tomar un puntero crudo hacia ellos y usarlo únicamente hasta terminar con el `Node` y querer destruirlo. Entonces podríamos hacer `take` del `Box` de `real_next` y eliminarlo. Creo que eso cumpliría con nuestro modelo muy simplificado de préstamos apilados.

Si quieres intentar hacerlo, que te "diviertas", pero tiene un aspecto horrible, ¿verdad? Este no es el capítulo sobre `Rc` y `RefCell`; ya no vamos a jugar a este *juego*. Simplemente haremos algo sencillo y limpio.

So instead we're going to use the very nice [Box::into_raw][] function:

> ```rust ,ignore
>   pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> Consumes the Box, returning a wrapped raw pointer.
>
> The pointer will be properly aligned and non-null.
>
>After calling this function, the caller is responsible for the memory previously managed by the Box. In particular, the caller should properly destroy T and release the memory, taking into account the memory layout used by Box. The easiest way to do this is to convert the raw pointer back into a Box with the `Box::from_raw` function, allowing the Box destructor to perform the cleanup.
>
> Nota: this is an associated function, which means that you have to call it as `Box::into_raw(b)` instead of `b.into_raw()`. This is so that there is no conflict with a method on the inner type.
>
> **Examples**
>
> Converting the raw pointer back into a Box with Box::from_raw for automatic cleanup:
>
> ```
>  let x = Box::new(String::from("Hello"));
>  let ptr = Box::into_raw(x);
>  let x = unsafe { Box::from_raw(ptr) };
> ```

Genial, parece *literalmente* diseñado para nuestro caso de uso. También encaja con las reglas que intentamos seguir: empezar con elementos seguros, convertirlos en punteros crudos y volver a convertirlos en elementos seguros únicamente al final, cuando queramos hacer `Drop`.

Esto es prácticamente igual que hacer lo de `real_next`, pero sin tener que complicarnos almacenando el `Box` cuando, de todos modos, contiene exactamente el mismo puntero que el puntero crudo.

Además, ahora que usamos punteros crudos en todas partes, no nos preocupemos por mantener pequeños los bloques `unsafe`: todo es inseguro. (Siempre lo fue, pero a veces está bien engañarse un poco.)


```rust ,ignore
pub fn push(&mut self, elem: T) {
    unsafe {
        // Immediately convert the Box into a raw pointer
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```


¡Ese código se ve mucho más limpio ahora que nos ceñimos a los punteros crudos!

Pasemos a `pop`, que también es bastante parecido a como lo dejamos, aunque debemos recordar usar `Box::from_raw` para liberar la asignación:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    unsafe {
        if self.head.is_null() {
            None
        } else {
            // RISE FROM THE GRAVE
            let head = Box::from_raw(self.head);
            self.head = head.next;

            if self.head.is_null() {
                self.tail = ptr::null_mut();
            }

            Some(head.elem)
        }
    }
}
```

Nuestros pequeños `take` y `map` han desaparecido; ahora solo tenemos que comprobar y establecer `null` manualmente.

And while we're here, let's slap in the destructor. This time we'll implement it as just repeatedly popping, because it's cute and simple:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```


Now, for the moment of truth:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

```text
cargo test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

Good, but does miri agree?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured
```

EYYYY!!!!!

IT FRIGGIN WORKED!

PROBABLY!

FAILING TO FIND UNDEFINED BEHAVIOUR IS NOT A PROOF THAT IT ISN'T THERE WAITING TO CAUSE PROBLEMS BUT THERE IS A LIMIT TO HOW RIGOROUS I AM WILLING TO BE FOR A JOKE BOOK ABOUT LINKED LISTS SO WE'RE GONNA CALL THIS A 100% MACHINE VERIFIED PROOF AND ANYONE WHO SAYS OTHERWISE CAN SUCK MY COQ!

∴ QED □


[Box::into_raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw
