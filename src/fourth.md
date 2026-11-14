# Una Doble Cola Mala Pero Segura Doblemente Enlazada

Ahora que hemos visto Rc y oído hablar de la mutabilidad interior, esto
plantea un pensamiento interesante... quizás *podemos* mutar a través de un
Rc. Y si ese *es* el caso, quizás podamos implementar una lista
*doblemente* enlazada totalmente segura.

En el proceso nos familiarizaremos con la *mutabilidad interior (interior
mutability)*, y probablemente aprenderemos por las malas que seguro no
significa *correcto*. Las listas doblemente enlazadas son difíciles, y siempre
cometo un error en alguna parte.

Añadamos un nuevo archivo llamado `fourth.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
```

Esta será otra operación en sala limpia, aunque como de costumbre
probablemente encontraremos alguna lógica que se aplica textualmente otra vez.

Descargo de responsabilidad: este capítulo es básicamente una demostración de
que esta es una muy mala idea.