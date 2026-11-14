# Cosas Extra (Extra Junk)

Ahora que hemos escrito `push` y `pop`, curiosamente todo lo demás es exactamente igual que en el caso de la pila. Solo las operaciones que cambian la longitud de la lista tienen que tocar el puntero `tail`.

Pero, por supuesto, ahora que todo son punteros inseguros tenemos que reescribir el código para usarlos. Y ya que vamos a tocar todo el código, podemos aprovechar para asegurarnos de que no nos hemos dejado nada.

En cualquier caso, empecemos copiando y pegando código de la implementación de la pila:

```rust ,ignore
// ...

pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

`IntoIter` parece correcto, pero `Iter` y `IterMut` incumplen nuestra regla sencilla de no usar punteros seguros en nuestros tipos. Seamos precavidos y cambiémoslos para que usen punteros crudos:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

¡Tiene buena pinta!

```text
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`
```

¡No tiene buena pinta! ¿Qué es eso de [`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) de lo que hablan?

> Zero-sized type used to mark things that “act like” they own a `T`.
>
> Adding a `PhantomData<T>` field to your type tells the compiler that your type acts as though it stores a value of type `T`, even though it doesn’t really. This information is used when computing certain safety properties.
>
> For a more in-depth explanation of how to use `PhantomData<T>`, please see [the Nomicon](https://doc.rust-lang.org/nightly/nomicon/).

Oye, no te precipites: estamos leyendo el libro que escribí *yo*, ¡no ese otro libro que seguramente escribió algún *empollón* enorme! Apuesto a que, si allí escriben una estructura de datos, será algo aburrido como una pila basada en arrays y *no* una lista enlazada.

> Unused lifetime parameters
>
> Perhaps the most common use case for PhantomData is a struct that has an unused lifetime parameter, typically as part of some unsafe code.

Ah, así que estamos declarando un tiempo de vida en nuestro tipo, pero en realidad no lo usamos. *Podríamos* seguir el camino de `PhantomData`, pero quiero reservarlo para la lista doblemente enlazada del capítulo siguiente, que *realmente* lo necesitará.

Estamos en una situación interesante en la que en realidad no necesitamos `PhantomData`. *Creo*. Voy a afirmarlo y confiar en que sea cierto; si al final `miri` nos grita, admitiré el error y usaremos `PhantomData`.

Lo que vamos a hacer en realidad es volver a poner las referencias en estos tipos de iterador y alegrarnos de que todavía podamos usarlas en algunos sitios. Creo que es correcto porque al usar un iterador sigue existiendo una especie de anidamiento adecuado: creas el iterador, utilizas referencias seguras durante un tiempo y después descartas el iterador.

Solo cuando desaparece el iterador puedes acceder a la lista y llamar a cosas como `push` y `pop`, que necesitan manipular el puntero `tail` y los `Box`. Durante la iteración *sí* vamos a desreferenciar un montón de punteros crudos, así que hay cierta mezcla, pero deberíamos poder considerar esas referencias como nuevos préstamos de los punteros inseguros.

*Ni siquiera yo* estoy convencido al 100 %, pero quiero probarlo y ver qué ocurre.

```rust ,ignore
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

Si vamos a almacenar referencias, tenemos que convertir nuestros punteros crudos en opciones de referencias. *Podríamos* comprobar si el puntero es nulo, pero este es uno de los casos excepcionalmente limitados en los que creo que está bien usar los desagradables métodos [`ptr::as_ref`](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) y [`ptr::as_mut`](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut).

*Normalmente* recomiendo evitar estos métodos como si fueran la peste, porque hacen cosas sorprendentes y desagradables y, por naturaleza, vuelven a introducir referencias cuando mi «regla sencilla» consiste precisamente en evitarlo.

Those methods come with a lot of warnings, but the most interesting is this:

> You must enforce Rust’s aliasing rules, since the returned lifetime `'a` is arbitrarily chosen and does not necessarily reflect the actual lifetime of the data. In particular, for the duration of this lifetime, the memory the pointer points to must not get accessed (read or written) through any other pointer.

Hey look it's the thing we talked about for 25 pages! I have already asserted we're *definitely* going to be fine to use references here, so aliasing solved! The other evil part is the signature:

```rust ,ignore
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

Do you see how that lifetime no es attached to the input at all, because `self` is by-value? Yeah that's what we call an "unbounded lifetime" and it's nasty stuff. Es willing to pretend to be as large as we ask it to be, even `'static`! The way you *deal* with that is by putting it somewhere that *is* bounded, which usually just means "return this from a function as soon as possible so that the function signature limits it".

Boy Estoy nervous about this but we're gonna keep pushing through! Vamos a steal some iterator impls from the stack:

```rust ,ignore
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.map(|node| {
                self.next = node.next.as_ref();
                &node.elem
            })
        }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            self.next.take().map(|node| {
                self.next = node.next.as_mut();
                &mut node.elem
            })
        }
    }
}
```

Moment of truth time...

```text
cargo test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test third::test::basics ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 15 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

YES!!! Take that **NARRATOR**! Sometimes I no make mistakes!

> **NARRATOR**: but no era the whole point that the mistakes are there to teach the reader.

YEAH WELL SOMETIMES THE LESSON IS THAT I'M RIGHT AND EVERYONE SHOULD LISTEN TO ME WHEN I SAY THINGS ABOUT UNSAFE CODE BECAUSE I HAVE SPENT FAR TOO MUCH TIME THINKING ABOUT THE SOUNDNESS OF ITERATOR IMPLEMENTATIONS?! OK?! OK.

Anyway here's `peek` and `peek_mut`.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref()
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut()
    }
}
```

Estoy not even gonna test them because I never make mistakes anymore.

> **NARRATOR**: `cargo build`

```text
error[E0308]: mismatched types
  --> src\fifth.rs:66:13
   |
25 | impl<T> List<T> {
   |      - this type parameter
...
64 |     pub fn peek(&self) -> Option<&T> {
   |                           ---------- expected `Option<&T>` 
   |                                      because of return type
65 |         unsafe {
66 |             self.head.as_ref()
   |             ^^^^^^^^^^^^^^^^^^ expected type parameter `T`, 
   |                                found struct `fifth::Node`
   |
   = note: expected enum `Option<&T>`
              found enum `Option<&fifth::Node<T>>`

```

FINE.

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

I guess I am going to *continue* to make mistakes, so we're going to be extra careful and add a new test Estoy going to call "miri food": something that just messes around and mixes up our APIs a bunch to help miri catch our mistakes.

```rust ,ignore
#[test]
fn miri_food() {
    let mut list = List::new();

    list.push(1);
    list.push(2);
    list.push(3);

    assert!(list.pop() == Some(1));
    list.push(4);
    assert!(list.pop() == Some(2));
    list.push(5);

    assert!(list.peek() == Some(&3));
    list.push(6);
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&30));
    assert!(list.pop() == Some(30));

    for elem in list.iter_mut() {
        *elem *= 100;
    }

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&400));
    assert_eq!(iter.next(), Some(&500));
    assert_eq!(iter.next(), Some(&600));
    assert_eq!(iter.next(), None);
    assert_eq!(iter.next(), None);

    assert!(list.pop() == Some(400));
    list.peek_mut().map(|x| *x *= 10);
    assert!(list.peek() == Some(&5000));
    list.push(7);

    // Drop it on the ground and let the dtor exercise itself
}
```


```text
cargo test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out



MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test

running 16 tests
test fifth::test::basics ... ok
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test fourth::test::peek ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::iter ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Perfect.
