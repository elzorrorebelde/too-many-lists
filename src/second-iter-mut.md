# IterMut

Voy a ser honesto, IterMut es SALVAJE. Lo cual en sí mismo parece una cosa
salvaje de decir; ¡seguramente es idéntico a Iter!

Semánticamente, sí, pero la naturaleza de las referencias compartidas y
mutables significa que Iter es "trivial" mientras que IterMut es Magia de
Hechicero Legítima.

La idea clave viene de nuestra implementación de Iterator para Iter:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

Que se puede des-azucarar a:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

¡La firma de `next` establece *ninguna* restricción entre el tiempo de vida
de la entrada y la salida! ¿Por qué nos importa? Significa que podemos llamar
a `next` una y otra vez incondicionalmente.


```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

¡Genial!

Esto es *definitivamente correcto* para referencias compartidas porque el
sentido mismo es que puedes tener montones de ellas a la vez. Sin embargo, las
referencias mutables *no pueden* coexistir. El sentido mismo es que son
exclusivas.

El resultado final es que es notablemente más difícil escribir IterMut usando
código seguro (y todavía no hemos entrado en qué significa eso...).
Sorprendentemente, ¡IterMut puede implementarse para muchas estructuras
completamente de forma segura!

Empezaremos simplemente tomando el código de Iter y cambiando todo para que
sea mutable:

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_deref_mut() }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_deref_mut() }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

Ok, parece que tenemos dos errores diferentes aquí. El primero parece muy
claro, ¡incluso nos dice cómo arreglarlo! No puedes mejorar una referencia
compartida a una mutable, así que `iter_mut` necesita tomar `&mut self`. Solo
un error tonto de copia-pega.

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_deref_mut() }
}
```

¿Y el otro?

¡Ups! En realidad cometí un error accidentalmente al escribir la impl de
`iter` en la sección anterior, y solo tuvimos suerte de que funcionara.

Acabamos de tener nuestro primer encuentro con la magia de Copy. Cuando
introdujimos la [propiedad (ownership)][ownership] dijimos que cuando mueves
cosas, ya no puedes usarlas más. Para algunos tipos, esto tiene perfecto
sentido. Nuestro buen amigo Box gestiona una asignación en el heap para
nosotros, y ciertamente no queremos que dos piezas de código piensen que
necesitan liberar su memoria.

Sin embargo, para otros tipos esto es *basura*. Los enteros no tienen
semántica de propiedad; son solo números sin significado. Por eso los enteros
están marcados como Copy. Los tipos Copy se sabe que son perfectamente
copiables mediante una copia a nivel de bits. Como tal, tienen un superpoder:
cuando se mueven, el valor anterior *sigue* siendo usable. Como consecuencia,
¡incluso puedes mover un tipo Copy fuera de una referencia sin reemplazo!

Todos los primitivos numéricos en Rust (i32, u64, bool, f32, char, etc...) son
Copy. También puedes declarar cualquier tipo definido por el usuario como Copy,
siempre que todos sus componentes sean Copy.

Críticamente para por qué este código funcionaba, ¡las referencias compartidas
también son Copy! Como `&` es copy, `Option<&>` *también* es Copy. Así que
cuando hicimos `self.next.map` estaba bien porque el Option simplemente se
copiaba. Ahora no podemos hacer eso, porque `&mut` no es Copy (si copiaras un
`&mut`, tendrías dos `&mut` a la misma ubicación en memoria, lo cual está
prohibido). En su lugar, deberíamos usar `take` para obtenerlo apropiadamente.


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_deref_mut();
        &mut node.elem
    })
}
```

```text
> cargo build

```

Uh... vaya. ¡Santo cielo! ¡IterMut Simplemente Funciona!

Probemos esto:


```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
    assert_eq!(iter.next(), None);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

Síp. Funciona.

Santo cielo.

Qué.

Ok, quiero decir que en realidad *se supone* que funciona, ¡pero normalmente
hay algo estúpido que se interpone! Seamos claros:

Acabamos de implementar una pieza de código que toma una lista enlazada
individualmente, y devuelve una referencia mutable a cada elemento de la lista
como máximo una vez. Y está verificada estáticamente para hacer eso. Y es
totalmente segura. Y no tuvimos que hacer nada salvaje.

Eso es algo importante, si me preguntas. Hay un par de razones por las que
esto funciona:

* Usamos `take` en el `Option<&mut>` para tener acceso exclusivo a la
  referencia mutable. No hay necesidad de preocuparse de que alguien la mire
  de nuevo.
* Rust entiende que está bien fragmentar una referencia mutable en los
  subcampos del struct al que apunta, porque no hay forma de "volver arriba",
  y están definitivamente separados.

Resulta que puedes aplicar esta lógica básica para obtener un IterMut seguro
para un array o un árbol también. ¡Incluso puedes hacer que el iterador sea
DoubleEnded, para que puedas consumir el iterador desde el frente *y* desde
atrás a la vez! ¡Guau!

[ownership]: first-ownership.md