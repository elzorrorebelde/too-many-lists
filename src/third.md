# Una Pila Persistente Enlazada Individualmente

Muy bien, hemos dominado el arte de las pilas enlazadas individualmente
mutables.

Pasemos de la propiedad *única (single ownership)* a la propiedad *compartida
(shared ownership)* escribiendo una lista enlazada individualmente *persistente*
e inmutable. Esta será exactamente la lista que los programadores funcionales
han llegado a conocer y amar. Puedes obtener la cabeza *o* la cola y poner la
cabeza de alguien en la cola de otro... y... eso es básicamente todo.
La inmutabilidad es una droga poderosa.

En el proceso, nos familiarizaremos en gran medida con Rc y Arc, pero esto
nos preparará para la siguiente lista que *cambiará el juego*.

Añadamos un nuevo archivo llamado `third.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
```

Sin copia-pega esta vez. Esta es una operación en sala limpia.