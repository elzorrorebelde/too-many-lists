# Una Pila Aceptable Enlazada Individualmente

En el capítulo anterior escribimos una pila enlazada individualmente mínima
viable. Sin embargo, hay algunas decisiones de diseño que la hacen algo
mala. Hagámosla menos mala. Al hacerlo, vamos a:

* Reinventar la rueda
* Hacer que nuestra lista pueda manejar cualquier tipo de elemento
* Añadir peek (ojear)
* Hacer nuestra lista iterable

Y en el proceso aprenderemos sobre

* Uso avanzado de Option
* Genéricos
* Tiempos de vida (lifetimes)
* Iteradores

Añadamos un nuevo archivo llamado `second.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
```

Y copiemos todo de `first.rs` en él.