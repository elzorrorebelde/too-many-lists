# Pruebas (Testing)

Muy bien, tenemos `push` y `pop` escritos, ¡ahora podemos probar nuestra
pila! Rust y cargo soportan pruebas como una característica de primera clase,
así que será super fácil. Todo lo que tenemos que hacer es escribir una
función, y anotarla con `#[test]`.

Generalmente, tratamos de mantener nuestras pruebas cerca del código que están
probando en la comunidad de Rust. Sin embargo, normalmente hacemos un nuevo
espacio de nombres para las pruebas, para evitar conflictos con el código
"real". Así como usamos `mod` para especificar que `first.rs` debería ser
incluido en `lib.rs`, podemos usar `mod` para básicamente crear un archivo
nuevo completo *en línea*:

```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

Y lo invocamos con `cargo test`.

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

¡Yupi, nuestra prueba que no hace nada pasó! Ahora hagamos que no sea una que
no hace nada. Haremos eso con la macro `assert_eq!`. Esto no es magia especial
de pruebas. Todo lo que hace es comparar las dos cosas que le das, y hacer
que el programa entre en pánico si no coinciden. ¡Sí, le indicas fallo al
sistema de pruebas volviéndote loco!

```rust ,ignore
mod test {
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
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

¡Ups! Como creamos un nuevo módulo, necesitamos importar List explícitamente
para usarlo.

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ADesires/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

¡Yupi!

¿Qué pasa con esa advertencia...? ¡Claramente usamos List en nuestra prueba!

...¡pero solo al probar! Para aplacar al compilador (y para ser amigables con
nuestros consumidores), deberíamos indicar que todo el módulo `test` solo
debería compilarse si estamos ejecutando pruebas.

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

¡Y eso es todo para las pruebas!