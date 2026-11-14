# Una Pila Mala Enlazada Individualmente

Este va a ser *con diferencia* el más largo, ya que necesitamos introducir básicamente
todo Rust, y vamos a construir algunas cosas "a la manera difícil" para entender
mejor el lenguaje.

Pondremos nuestra primera lista en `src/first.rs`. Necesitamos decirle a Rust que `first.rs` es
algo que nuestra lib usa. Todo lo que requiere es que pongamos esto al principio de
`src/lib.rs` (que Cargo creó por nosotros):

```rust ,ignore
// in lib.rs
pub mod first;
```