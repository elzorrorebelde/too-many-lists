# Drop

Como con las listas mutables, tenemos un problema de destructor recursivo.
Admitidamente, esto no es un problema tan grave para la lista inmutable: si
alguna vez alcanzamos otro nodo que es la cabeza de otra lista *en algún
lugar*, no lo soltaremos recursivamente. Sin embargo, sigue siendo algo que
debería importarnos, y cómo lidiar con ello no está tan claro. Así es como lo
resolvimos antes:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

El problema es el cuerpo del bucle:

```rust ,ignore
cur_link = boxed_node.next.take();
```

Esto está mutando el Node dentro del Box, pero no podemos hacer eso con Rc;
solo nos da acceso compartido, porque cualquier número de otros Rc podrían
estar apuntando a él.

Pero si supiéramos que somos la última lista que conoce este nodo,
*entonces* estaría bien mover el Node fuera del Rc. Entonces también podríamos
saber cuándo parar: cuando *no podamos* sacar el Node.

Y mira eso, Rc tiene un método que hace exactamente esto: `try_unwrap`:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

```text
cargo test
   Compiling lists v0.1.0 (/Users/ADesires/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ADesires/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

¡Genial!
Bien.