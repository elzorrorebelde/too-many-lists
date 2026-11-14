# La Lista Enlazada Asignada en la Pila

Este libro se centra principalmente en listas enlazadas *asignadas en el montón*, porque son las más comunes y prácticas, pero no *tenemos* por qué usar asignación en el montón. La asignación en el montón facilita reservar memoria dinámicamente. La asignación en la pila es menos amable en este sentido; cosas como `alloca` de C suelen considerarse Muy Malditas Y Problemáticas.

Así que asignemos memoria en la pila de la forma sencilla: llamando a una función y obteniendo un nuevo marco de pila con más espacio. Es una solución muy tonta a nuestro problema, pero también es realmente práctica y útil. Se hace constantemente, quizá sin pensar siquiera en ello como una lista enlazada.

Cada vez que haces algo de forma recursiva, puedes pasar un puntero al estado del paso actual al siguiente paso. Si ese puntero forma parte del propio estado, ¡has creado una lista enlazada asignada en la pila!

Por supuesto, ahora estamos en la parte *tonta* del libro, así que lo haremos de una forma tonta: convirtiendo la lista enlazada en la protagonista y obligando a todo el código del usuario a vivir en un pantano de callbacks. ¡A todo el mundo le encantan los callbacks anidados!

Nuestro tipo `List` será simplemente un `Node` con una referencia a otro `Node`:

```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a List<'a, T>>,
}
```

Solo tendrá una operación, `push`, que recibirá la lista antigua, el estado del nodo actual y un callback. El callback producirá la lista nueva. También permitiremos que los callbacks devuelvan cualquier valor, que `push` devolverá al terminar:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&List<'a, T>) -> U,
    ) -> U {
        let list = List { data, prev };
        callback(&list)
    }
}
```

¡Eso es todo! Podemos usarlo así:

```rust ,ignore
List::push(None, 3, |list| {
    println!("{}", list.data);
    List::push(Some(list), 5, |list| {
        println!("{}", list.data);
        List::push(Some(list), 13, |list| {
            println!("{}", list.data);
        })
    })
})
```

Es precioso. 😿

El usuario ya puede recorrer esta lista usando `while let` para avanzar por los valores de `prev`, pero, por diversión, implementemos un iterador, como de costumbre:

```rust ,ignore
impl<'a, T> List<'a, T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev;
            &node.data
        })
    }
}
```

Vamos a probarlo:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn elegance() {
        List::push(None, 3, |list| {
            assert_eq!(list.iter().copied().sum::<i32>(), 3);
            List::push(Some(list), 5, |list| {
                assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
                List::push(Some(list), 13, |list| {
                    assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
                })
            })
        })
    }
}
```

```text
> cargo test

running 18 tests
test fifth::test::into_iter ... ok
test fifth::test::iter ... ok
test fifth::test::iter_mut ... ok
test fifth::test::basics ... ok
test fifth::test::miri_food ... ok
test first::test::basics ... ok
test second::test::into_iter ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test second::test::iter_mut ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test second::test::peek ... ok
test third::test::iter ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

Llegados a este punto quizá te preguntes: «¿puedo modificar los datos almacenados en un nodo?». ¡Quizá! Intentemos hacer que la lista use referencias mutables en lugar de referencias compartidas:


```rust
pub struct List<'a, T> {
    pub data: T,
    pub prev: Option<&'a mut List<'a, T>>,
}

pub struct Iter<'a, T> {
    next: Option<&'a List<'a, T>>,
}

impl<'a, T> List<'a, T> {
    pub fn push<U>(
        prev: Option<&'a mut List<'a, T>>, 
        data: T, 
        callback: impl FnOnce(&mut List<'a, T>) -> U,
    ) -> U {
        let mut list = List { data, prev };
        callback(&mut list)
    }

    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: Some(self) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.prev.as_ref().map(|prev| &**prev);
            &node.data
        })
    }
}

```


```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:47:32
   |
46 |  List::push(Some(list), 13, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
47 |      assert_eq!(list.iter().copied().sum::<i32>(), 13 + 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:45:28
   |
44 |  List::push(Some(list), 5, |list| {
   |                             ----
   |                             |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |      assert_eq!(list.iter().copied().sum::<i32>(), 5 + 3);
   |                 ^^^^^^^^^^^ `list` escapes the closure body here


<ad infinitum>
```

Vaya. Parece que a nuestro iterador no le gusta esto. ¿Quizá lo hemos estropeado? Simplifiquemos un poco la prueba para comprobarlo:


```rust ,ignore
#[test]
fn elegance() {
    List::push(None, 3, |list| {
        assert_eq!(list.data, 3);
        List::push(Some(list), 5, |list| {
            assert_eq!(list.data, 5);
            List::push(Some(list), 13, |list| {
                assert_eq!(list.data, 13);
            })
        })
    })
}
```

```text
> cargo test

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:46:17
   |
44 |   List::push(Some(list), 5, |list| {
   |                              ----
   |                              |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
45 |       assert_eq!(list.data, 5);
46 | /     List::push(Some(list), 13, |list| {
47 | |         assert_eq!(list.data, 13);
48 | |     })
   | |______^ `list` escapes the closure body here

error[E0521]: borrowed data escapes outside of closure
  --> src\silly2.rs:44:13
   |
42 |   List::push(None, 3, |list| {
   |                        ----
   |                        |
   |              `list` declared here, outside of the closure body
   |              `list` is a reference that is only valid in the closure body
43 |       assert_eq!(list.data, 3);
44 | /     List::push(Some(list), 5, |list| {
45 | |         assert_eq!(list.data, 5);
46 | |         List::push(Some(list), 13, |list| {
47 | |             assert_eq!(list.data, 13);
48 | |         })
49 | |     })
   | |______________^ `list` escapes the closure body here
```

Mmm, no, esto sigue siendo un desastre.

El problema es que nuestra lista depende accidentalmente (😉) de la *varianza*. [La varianza es un tema complicado](https://doc.rust-lang.org/nomicon/subtyping.html), pero veámosla aquí de forma simplificada:

Cada lista contiene una referencia a una `List` con *exactamente el mismo tipo que ella*. Desde la perspectiva de la lista más interna, eso significa que todas las listas usan el mismo tiempo de vida que ella, pero esto es *objetivamente* falso: cada nodo de la lista vive estrictamente más tiempo que el siguiente, porque están literalmente en ámbitos anidados.

Entonces... ¿por qué compilaba el código cuando usábamos referencias compartidas? ¡Porque en muchos casos el compilador sabe que es seguro que algo viva «demasiado tiempo»! Cuando introducimos una referencia a una lista en la siguiente, el compilador «reduce» silenciosamente los tiempos de vida para que encajen con lo que espera la lista nueva. Esta reducción de tiempos de vida es la *varianza*.

Es exactamente el mismo truco que se usa en lenguajes con herencia para pasar un `Cat` donde se espera un `Animal` (un supertipo de `Cat`). Intuitivamente sabemos que está bien pasar un `Cat` cuando se espera un `Animal`, porque un `Cat` es un `Animal` *y algo más*. Está *bien* olvidar durante un rato la parte de «y algo más», ¿verdad?

Del mismo modo, un tiempo de vida más largo es simplemente un tiempo de vida más corto *y algo más*. Así que también está bien olvidar aquí la parte de «y algo más».

Pero, por supuesto, ahora te preguntarás: ¡¿por qué no funciona la versión con referencias mutables?!

Bueno, la varianza *no* siempre es segura. Si nuestro código *llegara* a compilar, podríamos escribir un uso después de liberar memoria como este:

```rust ,ignore
List::push(None, 3, |list| {
    List::push(Some(list), 5, |list| {
        List::push(Some(list), 13, |list| {
            // HAHAHA all the lifetimes are the same, so the compiler will
            // let me rewrite my parent to hold a mutable reference to myself!
            // I will create all the use-after-frees!!
            *list.prev.as_mut().unwrap().prev = Some(list);
        })
    })
})
```

El problema de olvidar detalles es que *en algún otro lugar podrían recordarse
esos detalles y esperar que sigan siendo ciertos*. Es un problema muy grave
cuando introduces la *mutabilidad*. Si no tienes cuidado, el código que no
recuerda la parte de «y algo más» que descartamos podría creer que está bien
escribir en lugares que la «recuerdan» y *esperan* que siga presente.

En términos de herencia, este código tiene que ser ilegal:

```rust ,ignore
let mut my_kitty = Cat;                  // Make a Cat (long lifetime)
let animal: &mut Animal = &mut my_kitty; // Forget it's a Cat (shorten lifetime)
*animal = Dog;                           // Write a Dog (short lifetime)
my_kitty.meow();                         // Meowing Dog! (Use After Free)
```

Así que puedes acortar el tiempo de vida de una referencia mutable, pero cuando
empiezas a *anidarlas* las cosas se vuelven «invariantes» y ya no puedes acortarlo.
lifetimes anymore. 

En concreto, `&mut &'big mut T` no se puede convertir en `&mut &'small mut T`,
donde `'big` es mayor que `'small`. O, de forma más formal, `&'a mut T` es
covariante respecto de `'a`, pero invariante respecto de `T`.

Dato curioso: Java permite explícitamente hacer este tipo de cosas, pero
[hace comprobaciones en tiempo de ejecución para evitar perros que maúllen](https://docs.oracle.com/javase/7/docs/api/java/lang/ArrayStoreException.html).

----

Entonces, ¿qué podemos hacer para modificar los datos? ¡Usar mutabilidad
interior! Esto permite indicar al compilador que solo queremos modificar los
*datos*, sin cambiar las referencias.

Podemos volver a la versión anterior del código, con referencias compartidas,
y usar `Cell` en una prueba nueva:

```rust ,ignore
#[test]
fn cell() {
    use std::cell::Cell;

    List::push(None, Cell::new(3), |list| {
        List::push(Some(list), Cell::new(5), |list| {
            List::push(Some(list), Cell::new(13), |list| {
                // Multiply every value in the list by 10
                for val in list.iter() {
                    val.set(val.get() * 10)
                }

                let mut vals = list.iter();
                assert_eq!(vals.next().unwrap().get(), 130);
                assert_eq!(vals.next().unwrap().get(), 50);
                assert_eq!(vals.next().unwrap().get(), 30);
                assert_eq!(vals.next(), None);
                assert_eq!(vals.next(), None);
            })
        })
    })
}
```

```text
> cargo test

running 19 tests
test fifth::test::into_iter ... ok
test fifth::test::basics ... ok
test fifth::test::iter_mut ... ok
test fifth::test::iter ... ok
test fourth::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test first::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fifth::test::miri_food ... ok
test silly2::test::cell ... ok
test third::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test silly1::test::walk_aboot ... ok
test silly2::test::elegance ... ok
test third::test::basics ... ok
test second::test::iter ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

Easy as recursive pie! ✨
