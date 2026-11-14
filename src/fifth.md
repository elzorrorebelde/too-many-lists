# Una Cola Insegura Aceptable Enlazada Individualmente

Ok, eso de la mutabilidad interior con contador de referencias se salió un
poco de control. Seguramente Rust no espera realmente que hagas ese tipo de
cosas en general? Bueno, sí y no. Rc y RefCell pueden ser geniales para
manejar casos simples, pero pueden volverse difíciles de manejar.
Especialmente si quieres ocultar que está sucediendo. ¡Tiene que haber una
mejor manera!

En este capítulo vamos a retroceder a listas enlazadas individualmente e
implementar una cola enlazada individualmente para mojarnos los pies con
*punteros crudos (raw pointers)* y *Rust Inseguro (Unsafe Rust)*.

> **NARRADOR:** Y señalaré los errores.

Y no cometeremos *ningún* error.

Añadamos un nuevo archivo llamado `fifth.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
pub mod fifth;
```

Nuestro código se derivará en gran medida de second.rs, ya que una cola es
principalmente un aumento de una pila en el mundo de las listas enlazadas.
Aún así, vamos a empezar desde cero porque hay algunos problemas
fundamentales que queremos abordar con el diseño y demás.