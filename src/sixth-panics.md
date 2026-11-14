# Drop y Seguridad ante Pánicos

Oye, ¿te fijaste en este comentario?

```rust
// Note that we don't need to mess around with `take` anymore
// because everything is Copy and there are no dtors that will
// run if we mess up... right? :) Riiiight? :)))
```

¿Es correcto?

Perdona, ¿has olvidado qué libro estás leyendo? ¡Por supuesto que está mal! (Más o menos.)

Veamos de nuevo el cuerpo interno de `pop_front`:

```rust ,ignore
// Bring the Box back to life so we can move out its value and
// Drop it (Box continues to magically understand this for us).
let boxed_node = Box::from_raw(node.as_ptr());
let result = boxed_node.elem;

// Make the next node into the new front.
self.front = boxed_node.back;
if let Some(new) = self.front {
    // Cleanup its reference to the removed node
    (*new.as_ptr()).front = None;
} else {
    // If the front is now null, then this list is now empty!
    debug_assert!(self.len == 1);
    self.back = None;
}

self.len -= 1;
result
// Box gets implicitly freed here, knows there is no T.
```

¿Ves el error? Lo aterrador es que está precisamente en esta línea:

```rust ,ignore
debug_assert!(self.len == 1);
```

¿¡En serio!? ¿¡Nuestra maldita comprobación de integridad para las pruebas es un error!? ¡Sí! Si implementamos bien nuestra colección *no debería* serlo, pero puede convertir algo benigno como «no estamos actualizando bien `len`» en *un error explotable de seguridad de memoria*. ¿Por qué? ¡Porque puede provocar un pánico! Normalmente no tienes que pensar ni preocuparte por los pánicos, pero cuando empiezas a escribir código *realmente* inseguro y a tomarte a la ligera las «invariantes», debes vigilar los pánicos con muchísimo cuidado.

Tenemos que hablar de la [*seguridad frente a excepciones*](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html) (también llamada seguridad frente a pánicos o seguridad durante el desenrollado).

La cuestión es la siguiente: de forma predeterminada, los pánicos provocan un *desenrollado*. «Desenrollado» es una forma elegante de decir «haz que todas las funciones retornen inmediatamente». Podrías pensar «si *todo el mundo* retorna, el programa está a punto de morir, así que ¿por qué debería importarnos?», pero te equivocarías.

Debemos preocuparnos por dos motivos: los destructores se ejecutan cuando una función retorna y el desenrollado puede *capturarse*. En ambos casos, el código puede seguir ejecutándose después de un pánico, así que debemos asegurarnos de que nuestras colecciones inseguras siempre estén en un estado coherente cuando pueda producirse un pánico, porque cada pánico es un retorno anticipado implícito.

Pensemos en el estado en el que se encuentra nuestra colección cuando llegamos a esa línea:

We have our boxed_node on the stack, and we've extracted the element from it. If we were to return at this point, the Box would be dropped, and the node would be freed. Do you see it now..? self.back is still pointing at that freed node! Once we implement the rest of our collection and start using self.back for things, this could result in a use-after-free! Yikes!

Curiosamente, esta línea tiene problemas parecidos, pero es mucho más segura:

```rust ,ignore
self.len -= 1;
```

De forma predeterminada, Rust comprueba los desbordamientos inferiores y superiores en las compilaciones de depuración y provoca un pánico cuando suceden. ¡Sí, toda operación aritmética es un riesgo para la seguridad frente a pánicos! Esta es *mejor* porque ocurre después de reparar todas nuestras invariantes, así que no causa problemas de seguridad de memoria... siempre que no confiemos en que `len` sea correcto. Pero si se produce un desbordamiento inferior, sin duda hay un error y estaríamos perdidos igualmente. La aserción de depuración es, en cierto sentido, *peor*, porque puede convertir un problema menor en uno crítico.

He mencionado varias veces el término «invariantes» porque es un concepto muy útil para la seguridad frente a pánicos. Básicamente, desde el punto de vista de un observador externo, hay ciertas propiedades que nuestra colección siempre mantiene. En una `LinkedList`, una de ellas es que cualquier nodo accesible desde la lista sigue estando asignado e inicializado.

*Inside* the implementation we have a bit more flexibility to break invariants *temporarily* as long as we make sure to repair them *before anyone notices*. This is actually one of the "killer apps" of Rust's ownership and borrowing system for collections: if an operation requires an `&mut Self`, then we are *guaranteed* that we have exclusive access to our collection and that it's fine for us to temporarily break invariants, safe in the knowledge that no one can sneakily mess with it.

Perhaps the greatest expression of this is [Vec::drain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.drain), which actually lets you completely smash a core invariant of Vec and start moving values out from the *front* or even *middle* of a Vec. The reason this is *sound* is because the Drain iterator that we return holds an `&mut` to the Vec, and so all access is gated behind it! No one can observe the Vec until the Drain iterator goes away, and then it's destructor can "repair" the Vec before anyone can notice, it's perfe--

[Es not perfect](https://doc.rust-lang.org/nightly/nomicon/leaking.html#drain). Desafortunadamente, you [no puede rely on destructors in code you no control to run](https://doc.rust-lang.org/std/mem/fn.forget.html), and so even with Drain we need to do a little extra work to make our type always preserved invariants, but in a kind of goofy way: [we just set the Vec's len to 0 at the start](https://doc.rust-lang.org/std/mem/fn.forget.html), so if anyone leaks the Drain, then they will have a *safe* Vec... but they will have also lost a bunch of data. You leak me? I leak you! An eye for an eye! True justice!

For a situation where you *can* actually use destructors for panic-safety, check out the [BinaryHeap::sift_up case study](https://doc.rust-lang.org/nightly/nomicon/exception-safety.html#binaryheapsift_up).

En cualquier caso, no necesitaremos toda esta sofisticación para nuestras `LinkedList`; solo tenemos que vigilar un poco más dónde rompemos nuestras invariantes, qué damos por correcto y qué exigimos que sea correcto, y evitar introducir desenrollados innecesarios en mitad de tareas complicadas.

In this case, we have two options to make our code a bit more robust:

* Usar operaciones como `Option::take` de forma mucho más agresiva, porque son más «transaccionales» y tienden a preservar las invariantes.

* Eliminar las `debug_assert` y confiar en que escribiremos mejores pruebas mediante funciones de «comprobación de integridad» que nunca se ejecuten en el código del usuario.

In principle I like the first option, but it no actually work great for a doubly-linked list, because everything is doubly-redundantly encoded. Option::take no fix the problem here, but moving the debug_assert down a line would. But really, why make things harder for ourselves? Simplemente eliminemos esas `debug_assert` y asegurémonos de que todo lo que pueda provocar un pánico esté al principio o al final de nuestros métodos, donde sabemos que nuestras invariantes deberían cumplirse.

(In this way it's perhaps more accurate to think of them as *preconditions* and *postconditions* but you really should endeavour to treat them as invariants as much as possible!)

Here's our full implementation now:

```rust
use std::ptr::NonNull;
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    _boo: PhantomData<T>,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            front: None,
            back: None,
            len: 0,
            _boo: PhantomData,
        }
    }

    pub fn push_front(&mut self, elem: T) {
        // SAFETY: it's a linked-list, what do you want?
        unsafe {
            let new = NonNull::new_unchecked(Box::into_raw(Box::new(Node {
                front: None,
                back: None,
                elem,
            })));
            if let Some(old) = self.front {
                // Put the new front before the old one
                (*old.as_ptr()).front = Some(new);
                (*new.as_ptr()).back = Some(old);
            } else {
                // If there's no front, then we're the empty list and need 
                // to set the back too.
                self.back = Some(new);
            }
            // These things always happen!
            self.front = Some(new);
            self.len += 1;
        }
    }

    pub fn pop_front(&mut self) -> Option<T> {
        unsafe {
            // Only have to do stuff if there is a front node to pop.
            self.front.map(|node| {
                // Bring the Box back to life so we can move out its value and
                // Drop it (Box continues to magically understand this for us).
                let boxed_node = Box::from_raw(node.as_ptr());
                let result = boxed_node.elem;

                // Make the next node into the new front.
                self.front = boxed_node.back;
                if let Some(new) = self.front {
                    // Cleanup its reference to the removed node
                    (*new.as_ptr()).front = None;
                } else {
                    // If the front is now null, then this list is now empty!
                    self.back = None;
                }

                self.len -= 1;
                result
                // Box gets implicitly freed here, knows there is no T.
            })
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }
}
```

What can panic here? Well, knowing that honestly requires you to be a bit of a Rust expert, but thankfully, I am!

Los únicos lugares de este código que *podrían* provocar un pánico (salvo que alguien haga la barbaridad de recompilar la biblioteca estándar con `debug_assert` activadas, algo que nunca deberías hacer) son `Box::new` (por falta de memoria) y la aritmética de `len`. Todo eso está al principio o al final de nuestros métodos, así que sí, estamos siendo razonablemente seguros.

...¿te sorprendió que `Box::new` pudiera provocar un pánico? ¡Los pánicos son así! Intenta preservar esas invariantes para no tener que preocuparte por ello.

