# Una Introducción a los Cursores

¡¡¡Vale!!! Ahora tenemos una `LinkedList` a la altura de la implementación 1.0 de `std`. Lo que, por supuesto, significa que nuestra `LinkedList` *sigue siendo completamente inútil*. Hemos pagado la enorme penalización de rendimiento de implementar una `Deque` como lista enlazada, **y todavía no tenemos ninguna de las API que la harían realmente útil**.

Así nos va frente a los «casos de uso estrella» de las listas enlazadas:

* 🚫 Poder hacer [cosas intrusivas extrañas](https://docs.rs/linked-hash-map/latest/linked_hash_map/)
* 🚫 Poder hacer [cosas extrañas sin bloqueo](https://doc.rust-lang.org/std/sync/mpsc/)
* 🚫 Poder almacenar [tipos de tamaño dinámico](https://doc.rust-lang.org/nomicon/exotic-sizes.html#dynamically-sized-types-dsts)
* 🌟 `push`/`pop` en O(1) sin [amortización](https://en.wikipedia.org/wiki/Amortized_analysis) (si estás dispuesto a creer que `malloc` es O(1))
* 🚫 Dividir listas en O(1)
* 🚫 Intercalar listas en O(1)

Bueno... 1 de 6 es... ¡mejor que nada! ¿Ves por qué quería sacar esta cosa de `std`?

No vamos a hacer que nuestra lista admita cosas «extrañas», porque todo eso es específico de cada dominio y bastante improvisado. Pero lo de dividir e intercalar listas sí es algo que podemos hacer.

Pero aquí está el problema: *alcanzar* el elemento k<sup>ésimo</sup> de una `LinkedList` tarda O(k), así que ¿cómo podemos hacer *posibles* divisiones y fusiones arbitrarias en O(1)? El truco consiste en no tener una API como `split_at(index)`, sino en crear un sistema que permita al usuario iterar con estado hasta una posición de la lista y hacer modificaciones O(1) en ese punto.

Oye, ¡ya tenemos iteradores! ¿Podemos usarlos para esto? Más o menos... pero una de sus superpoderes se interpone. Quizá recuerdes que la forma en que escribimos los tiempos de vida de los iteradores por referencia hace que las referencias que devuelven *no* estén vinculadas al iterador. Esto nos permite llamar repetidamente a `next` y conservar los elementos:

```rust ,ignore
let mut list = ...;
let iter = list.iter_mut();
let elem1 = list.next();
let elem2 = list.next();

if elem1 == elem2 { ... }
```

Si las referencias devueltas tomaran prestado el iterador, este código no funcionaría en absoluto. ¡El compilador se quejaría de la segunda llamada a `next`! Esta flexibilidad es estupenda, pero nos impone algunas restricciones implícitas:

* Los iteradores por referencia mutable nunca pueden retroceder y devolver de nuevo un elemento, porque el usuario podría obtener dos `&mut` del mismo elemento, infringiendo reglas fundamentales del lenguaje.

* Los iteradores por referencia no pueden tener métodos adicionales que puedan modificar la colección subyacente de una forma que invalide una referencia ya devuelta.

¡Desafortunadamente, ambas cosas son *exactamente* lo que queremos que haga nuestra API de `LinkedList`! Por tanto, no podemos usar simplemente iteradores; necesitamos algo nuevo: los *cursores*.

Los cursores son como la pequeña barra parpadeante `|` que aparece al editar texto en un ordenador. Son una posición dentro de una secuencia (el texto) que puedes desplazar con las teclas de dirección; cada vez que escribes, la edición sucede en ese punto.

Mira: si

pulso

Intro,

todo el

texto

se parte por la mitad.

Perdona, estás detrás de mí observando cómo escribo esto, ¿verdad? Así que tiene todo el sentido, ¿no? ¿Verdad?

Ahora bien, si alguna vez tuviste la desgracia de usar un teclado con una tecla «Insert» y llegaste a pulsarla, sabrás que técnicamente hay dos interpretaciones de los cursores: pueden situarse entre elementos (caracteres) o *sobre* los elementos. Estoy bastante seguro de que nadie ha pulsado «Insert» a propósito en su vida y de que existe únicamente como Botón del Sufrimiento, así que resulta obvio cuál es la opción Mejor y Correcta: ¡los cursores se sitúan entre los elementos!

Una lógica inexpugnable, ¿verdad? No creo que nadie pueda llevarme la contraria.

¿Perdona? ¿Hubo una [RFC en 2018 para añadir cursores a la `LinkedList` de Rust](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md)?

> With a Cursor one can seek back and forth through a list and get the current element. With a CursorMut One can seek back and forth and get mutable references to elements, and it can insert and delete elements before and behind the current element (along with performing several list operations such as splitting and splicing).

*Current element*? This cursor is *on* elements, not between them! ¡No puedo creer que no aceptaran mi argumento totalmente irrefutable! Así que puedes usar el `Cursor` de `std`... espera, ¿estamos en [2023 y `Cursor` sigue marcado como inestable en Rust 1.71](https://doc.rust-lang.org/1.71.0/std/collections/linked_list/struct.CursorMut.html)?

Hey wait:

> Cursors always rest between two elements in the list, and index in a logically circular way. To accommodate this, there is a "ghost" non-element that yields None between the head and tail of the list.

HEY WAIT. This is the opposite of what the RFC says??? But wait all the docs on the methods still refer to "current" elements... wait hold on, where have I seen this ghost stuff before. Oh wait, no I do that in [my old linked-list fork](https://docs.rs/linked-list/0.0.3/linked_list/struct.Cursor.html) where I prototyped?

> Cursors always rest between two elements in the list, and index in a logically circular way. To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.

Un momento, ¿qué demonios? Esto no es una broma: ahora mismo estoy intentando leer la documentación de verdad. ¿Acaso `std` propuso mediante una RFC un diseño distinto del que presenté en 2015, pero después copió y pegó la documentación de mi prototipo? ¿`std` se está burlando de mí por escribir un libro sobre cuánto odio `LinkedList`? Sí, construí aquel prototipo para demostrar el concepto y conseguir que me dejaran añadirlo a `std` para que `LinkedList` dejara de ser inútil, pero... ¿qué demonios?

Ok you know what, clearly std is blessing my design as the objectively superior one, so we're going to do my design. Also that's nice because this entire chapter is me actually literally rewriting that library from scratch, so not changing the API sounds Good To Me!

Here's the full top-level docs I wrote:

> A Cursor is like an iterator, except that it can freely seek back-and-forth, and can safely mutate the list during iteration. This is because the lifetime of its yielded references are tied to its own lifetime, instead of just the underlying list. This means cursors cannot yield multiple elements at once.
>
> Cursors always rest between two elements in the list, and index in a logically circular way. To accomadate this, there is a "ghost" non-element that yields None between the head and tail of the List.
>
> When created, cursors start between the ghost and the front of the list. That is, next will yield the front of the list, and prev will yield None. Calling prev again will yield the tail.

Qué bonito. Aunque concluimos que todo lo del «nodo centinela» daba más problemas de los que resolvía, acabaremos teniendo una semántica que «finge» que hay un nodo centinela para que el cursor pueda rodear la lista y llegar al otro extremo.

*Skims over my old APIs some more*

```rust ,ignore
fn splice(&mut self, other: &mut LinkedList<T>)
```

> Inserts the entire list's contents right after the cursor.

Oh yeah, this is coming back to me. I wrote this when I was really mad about combinatoric explosion, and was trying to come up with a way for there to only be one copy of each operation. Unfortunately this is... semantically problematic. See, when the user wants to splice one list into another, they might want the cursor to end up *before* the splice or *after it*. The inserted list can be arbitrarily large, so it's a genuine issue for us to only allow for one and expect the user to walk over the entire inserted list!

Al final tendremos que rediseñar esto desde cero. ¿Qué necesita nuestro tipo `Cursor`? Necesita:

* point "between" two elements
* as a nice little feature, keep track of what "index" is next
* update the list itself to modify front/back/len. 

How do you point between two elements? Well, you don't. You just point at the "next" element. So, yeah even though we're exposing "cursor goes in-between" semantics, we're really implementing it as "cursor is on", and just pretending everything happens before or after that point.

¡Pero hay una razón! El caso de uso de `splice` quiere permitir que el usuario elija si termina antes o después de la lista, pero esto es... *horriblemente* complicado de expresar con la API de `std`. They have splice_after and splice_before, but neither changes the cursor's position, so really you'd need splice_after_before and splice_after_after...

Espera, estoy diciendo una tontería. En la API de `std` puedes elegir el nodo en el que quieres terminar y usar después `splice_after` o `splice_before`, según corresponda.

*squints*

Wait is the std API actually good.

*skims through the code*

Ok the std API is actually good.

Alright screw it, we're going to [implement the RFC](https://github.com/rust-lang/rfcs/blob/master/text/2570-linked-list-cursors.md). Or at least the interesting parts of it.

I have my quibbles with some of the terminology std uses, but cursors are always going to be a bit brain-melty: `iter().next_back()`  gets you `back()`, so that's good, but then each subsequent `next_back()` is actually bringing you *closer to the front* and indeed, every pointer we follow is a "front" pointer! If I think about this seeming-paradox too much it hurts my brain, so, I can certainly respect going for different terminology to avoid this.

La API de `std` habla de operaciones «antes» (hacia el frente) y «después» (hacia atrás), y en lugar de `next` y `next_back` llama a las operaciones `move_next` y `move_prev`. Mmm. Está tomando parte de la terminología de los iteradores, pero al menos `next` no evoca el frente ni la parte trasera y ayuda a orientarse respecto a cómo se comportan los iteradores.

We can work with this.
