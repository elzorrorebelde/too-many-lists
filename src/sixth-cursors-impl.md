# Implementando Cursores

Solo vamos a ocuparnos del `CursorMut` de `std`, porque la versión inmutable no es especialmente interesante. Al igual que en mi diseño original, tiene un elemento «fantasma» que contiene `None` para indicar el principio o el final de la lista, y se puede «pasar por encima» de él para rodear la lista y llegar al otro extremo. Para implementarlo necesitaremos:

* Un puntero al nodo actual
* Un puntero a la lista
* El índice actual

Espera, ¿cuál es el índice cuando apuntamos al «fantasma»?

*frunce el ceño* ... *consulta `std`* ... *no le gusta la respuesta de `std`*

Así que, razonablemente, `index` en un cursor devuelve un `Option<usize>`. La implementación de `std` hace un montón de cosas para evitar almacenarlo como `Option`, pero... nosotros tenemos una lista enlazada, no pasa nada. `std` también tiene `cursor_front` y `cursor_back`, que colocan el cursor en los elementos delantero y trasero; resulta intuitivo, pero luego hay que hacer algo extraño cuando la lista está vacía.

Puedes implementar todo eso si quieres, pero yo voy a reducir la repetición y los casos límite y crear únicamente un método `cursor_mut` básico que empiece en el fantasma. Después, cada usuario podrá usar `move_next` o `move_prev` para llegar al elemento que quiera (y envolverlo en `cursor_front` si de verdad lo desea).

Vamos a ponernos manos a la obra:

```rust ,ignore
pub struct CursorMut<'a, T> {
    cur: Link<T>,
    list: &'a mut LinkedList<T>,
    index: Option<usize>,
}
```

Bastante sencillo: un campo por cada elemento de nuestra lista. Ahora, el método `cursor_mut`:

```rust ,ignore
impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}
```

Como empezamos en el fantasma, podemos inicializarlo todo a `None`; ¡sencillo! Pasemos al movimiento:


```rust ,ignore
impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }
}
```

Así que hay 4 casos interesantes:

* El caso normal.
* El caso normal, pero llegamos al fantasma.
* El caso del fantasma, en el que vamos al principio de la lista.
* El caso del fantasma, pero la lista está vacía, así que no hacemos nada.

`move_prev` sigue exactamente la misma lógica, pero invirtiendo `front`/`back` y los cambios del índice:

```rust ,ignore
pub fn move_prev(&mut self) {
    if let Some(cur) = self.cur {
        unsafe {
            // We're on a real element, go to its previous (front)
            self.cur = (*cur.as_ptr()).front;
            if self.cur.is_some() {
                *self.index.as_mut().unwrap() -= 1;
            } else {
                // We just walked to the ghost, no more index
                self.index = None;
            }
        }
    } else if !self.list.is_empty() {
        // We're at the ghost, and there is a real back, so move to it!
        self.cur = self.list.back;
        self.index = Some(self.list.len - 1)
    } else {
        // We're at the ghost, but that's the only element... do nothing.
    }
}
```

Ahora añadamos métodos para consultar los elementos alrededor del cursor: `current`, `peek_next` y `peek_prev`. **Nota muy importante:** estos métodos deben tomar prestado nuestro cursor mediante `&mut self`, y los resultados deben quedar vinculados a ese préstamo. No podemos permitir que el usuario obtenga varias copias de una referencia mutable ni que use nuestras API de `insert`, `remove`, `split` o `splice` mientras conserve una referencia de ese tipo.

Afortunadamente, esta es la suposición predeterminada que hace Rust al omitir los tiempos de vida, así que por defecto hará lo correcto.

```rust ,ignore
pub fn current(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur.map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_next(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).back)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}

pub fn peek_prev(&mut self) -> Option<&mut T> {
    unsafe {
        self.cur
            .and_then(|node| (*node.as_ptr()).front)
            .map(|node| &mut (*node.as_ptr()).elem)
    }
}
```

La cabeza vacía, los métodos de `Option` y los errores del compilador (omitidos) hacen ahora todo el trabajo mental. Era escéptico con lo de `Option<NonNull>`, pero, demonios, realmente me permite escribir este código en piloto automático. He pasado demasiado tiempo escribiendo colecciones basadas en arrays donde nunca puedes usar `Option`; ¡esto es maravilloso! (`(*node.as_ptr())` sigue siendo horrible, pero así son los punteros crudos de Rust.)

Ahora tenemos una elección: podemos saltar directamente a `split` y `splice`, que son el objetivo principal de estas API, o dar un pequeño paso implementando la inserción y eliminación de un solo elemento. Tengo la sensación de que acabaremos implementando `insert` y `remove` en términos de `split` y `splice`, así que hagamos primero estas últimas y veamos qué ocurre (sinceramente, no tengo ni idea mientras escribo esto).




# Split

Primero, `split_before` y `split_after`, que devuelven todo lo que hay antes o después del elemento actual como una `LinkedList` (se detienen en el elemento fantasma; si el cursor está en el fantasma, devuelven la lista completa y el cursor pasa a apuntar a una lista vacía):

*entrecierra los ojos* De acuerdo, aquí hay lógica no trivial, así que tendremos que analizarla paso a paso.

Veo 4 casos potencialmente interesantes para `split_before`:

* The normal case
* El caso normal, pero `prev` es el fantasma.
* El caso del fantasma, en el que devolvemos la lista completa y nos quedamos vacíos.
* El caso del fantasma, pero la lista está vacía, así que no hacemos nada y devolvemos una lista vacía.

Empecemos por los casos límite. Creo que el tercero se reduce simplemente a

```rust
mem::replace(self.list, LinkedList::new())
```

¿Verdad? Nos quedamos vacíos, devolvemos la lista completa y nuestros campos ya eran `None`, así que no hay nada que actualizar. Genial. ¡Y esto también hace lo correcto en el cuarto caso!

Pasemos ahora a los casos normales... de acuerdo, voy a necesitar algunos diagramas ASCII. En el caso más general tenemos algo como esto:

```text
list.front -> A <-> B <-> C <-> D <- list.back
                          ^
                         cur
```

Y queremos producir esto:

```text
list.front -> C <-> D <- list.back
              ^
             cur

return.front -> A <-> B <- return.back
```

Tenemos que romper el enlace entre `cur` y `prev`, y... Dios mío, hay que cambiar muchísimas cosas. De acuerdo, voy a dividirlo en pasos para convencerme de que tiene sentido. Será algo prolijo, pero al menos podré entenderlo:

```rust ,ignore
pub fn split_before(&mut self) -> LinkedList<T> {
    if let Some(cur) = self.cur {
        // We are pointing at a real element, so the list is non-empty.
        unsafe {
            // Current state
            let old_len = self.list.len;
            let old_idx = self.index.unwrap();
            let prev = (*cur.as_ptr()).front;
            
            // What self will become
            let new_len = old_len - old_idx;
            let new_front = self.cur;
            let new_back = self.list.back;
            let new_idx = Some(0);

            // What the output will become
            let output_len = old_len - new_len;
            let mut output_front = self.list.front;
            let output_back = prev;

            // Break the links between cur and prev
            if let Some(prev) = prev {
                (*cur.as_ptr()).front = None;
                (*prev.as_ptr()).back = None;
            } else {
                // We're at the first node, need to unset the head we "optimistically" set before.
                output_front = None;
            }

            // Produce the result:
            self.list.len = new_len;
            self.list.front = new_front;
            self.list.back = new_back;
            self.index = new_idx;

            LinkedList {
                front: output_front,
                back: output_back,
                len: output_len,
                _boo: PhantomData,
            }
        }
    } else {
        // We're at the ghost, just replace our list with an empty one.
        // No other state needs to be changed.
        std::mem::replace(self.list, LinkedList::new())
    }
}
```

Nótese que this if-let is handling the "normal case, but prev is the ghost" situation:

```rust ,ignore
if let Some(prev) = prev {
    (*cur.as_ptr()).front = None;
    (*prev.as_ptr()).back = None;
} else {
    output_front = None;
}
```

Si *quieres*, puedes juntar todo eso y aplicar optimizaciones como:

* convertir los dos accesos a `(*cur.as_ptr()).front` en un único `(*cur.as_ptr()).front.take()`
* observar que `new_back` no hace nada y eliminar ambas cosas

Por lo que puedo ver, todo lo demás hace casualmente lo correcto. ¡Ya lo veremos al escribir las pruebas! (Copia y pega para crear `split_after`.)

He terminado de cometer errores y ahora voy a intentar escribir el código más resistente a errores que pueda. Así es como *realmente* escribo colecciones: divido las cosas en pasos y casos triviales hasta que caben en mi cabeza y parecen infalibles. Después escribo un montón de pruebas hasta convencerme de que no he conseguido estropearlo.

Como la mayor parte del trabajo que he hecho con colecciones es *extremadamente inseguro*, normalmente no puedo confiar en que el compilador detecte los errores; además, ¡`miri` ni siquiera existía entonces! Así que tengo que mirar un problema hasta que me duela la cabeza e intentar con todas mis fuerzas No Cometer Jamás Un Error.

¡¡¡No escribas código Rust inseguro! El Rust seguro es muchísimo mejor!!!




# Splice

Solo queda un jefe más: `splice_before` y `splice_after`, que espero que sean los campeones de los casos límite. Las dos funciones *reciben* una `LinkedList` y injertan su contenido en la nuestra. Nuestra lista podría estar vacía, la otra también y además tenemos fantasmas... *suspira*; vayamos paso a paso con `splice_before`.

* Si su lista está vacía, no tenemos que hacer nada.
* Si nuestra lista está vacía, pasa a ser la lista recibida.
* Si apuntamos al fantasma, añadimos al final (cambiamos `list.back`).
* Si apuntamos al primer elemento (0), añadimos al principio (cambiamos `list.front`).
* En el caso general, hacemos un montón de manipulación de punteros.

El caso general es el siguiente:

```text
input.front -> 1 <-> 2 <- input.back

 list.front -> A <-> B <-> C <- list.back
                     ^
                    cur
```

Becoming this:

```text
list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
```

¿De acuerdo? De acuerdo. Escribámoslo... *TOMA UNA ENORME BOCANADA DE AIRE Y SE LANZA*:

```rust ,ignore
    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        unsafe {
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                if let Some(0) = self.index {
                    // We're appending to the front, see append to back
                    (*cur.as_ptr()).front = input.back.take();
                    (*input.back.unwrap().as_ptr()).back = Some(cur);
                    self.list.front = input.front.take();

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                } else {
                    // General Case, no boundaries, just internal fixups
                    let prev = (*cur.as_ptr()).front.unwrap();
                    let in_front = input.front.take().unwrap();
                    let in_back = input.back.take().unwrap();

                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);

                    // Index moves forward by input length
                    *self.index.as_mut().unwrap() += input.len;
                    self.list.len += input.len;
                    input.len = 0;
                }
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                // We can either `take` the input's pointers or `mem::forget`
                // it. Using take is more responsible in case we do custom
                // allocators or something that also needs to be cleaned up!
                (*back.as_ptr()).back = input.front.take();
                (*input.front.unwrap().as_ptr()).front = Some(back);
                self.list.back = input.back.take();
                self.list.len += input.len;
                // Not necessary but Polite To Do
                input.len = 0;
            } else {
                // We're empty, become the input, remain on the ghost
                *self.list = input;
            }
        }
    }
```

Ok this one is genuinely horrendous, and really is feeling that `Option<NonNull>` pain now. But hay a lot of cleanups we can do. For one, we can pull this code out to the very end, because we always want to do it. I no *love*  (although sometimes it's a noop, and setting `input.len` is more a matter of paranoia about future extensions to the code):

```rust ,ignore
self.list.len += input.len;
input.len = 0;
```

> Use of moved value: `input`

Ah, cierto: en el caso en que estamos vacíos estamos moviendo la lista. Sustituyamos eso por un intercambio:

```rust ,ignore
// We're empty, become the input, remain on the ghost
std::mem::swap(self.list, &mut input);
```

En este caso las escrituras serán innecesarias, pero siguen funcionando (probablemente también podríamos retornar pronto en esta rama para satisfacer al compilador).

Este `unwrap` es consecuencia de haber pensado en los casos al revés y se puede arreglar haciendo que el `if let` plantee la pregunta correcta:

```rust ,ignore
if let Some(0) = self.index {

} else {
    let prev = (*cur.as_ptr()).front.unwrap();
}
```

El ajuste del índice está duplicado dentro de las ramas, así que también podemos sacarlo fuera:

```rust
*self.index.as_mut().unwrap() += input.len;
```

De acuerdo, juntándolo todo obtenemos esto:

```rust
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        let in_front = input.front.take().unwrap();
        let in_back = input.back.take().unwrap();

        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // We're appending to the front, see append to back below
        (*cur.as_ptr()).front = input.back.take();
        (*input.back.unwrap().as_ptr()).back = Some(cur);
        self.list.front = input.front.take();
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    // We can either `take` the input's pointers or `mem::forget`
    // it. Using take is more responsible in case we do custom
    // allocators or something that also needs to be cleaned up!
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
    self.list.back = input.back.take();

} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

Alright this still sucks, but mostly because of -- nope ok just spotted a bug:

```rust
    (*back.as_ptr()).back = input.front.take();
    (*input.front.unwrap().as_ptr()).front = Some(back);
```

We `take` input.front and then unwrap it on the next line! *sigh* and we do the same thing in the equivalent mirror case. We would have caught this instantly in tests, but, we're trying to be Perfect now, and Estoy just kinda doing this live, and this is the exact moment where I saw it. This is what I get for not being my usual tedious self and doing things in phases. More explicit!

```rust
// We can either `take` the input's pointers or `mem::forget`
// it. Using `take` is more responsible in case we ever do custom
// allocators or something that also needs to be cleaned up!
if input.is_empty() {
    // Input is empty, do nothing.
} else if let Some(cur) = self.cur {
    // Both lists are non-empty
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    if let Some(prev) = (*cur.as_ptr()).front {
        // General Case, no boundaries, just internal fixups
        (*prev.as_ptr()).back = Some(in_front);
        (*in_front.as_ptr()).front = Some(prev);
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
    } else {
        // No prev, we're appending to the front
        (*cur.as_ptr()).front = Some(in_back);
        (*in_back.as_ptr()).back = Some(cur);
        self.list.front = Some(in_front);
    }
    // Index moves forward by input length
    *self.index.as_mut().unwrap() += input.len;
} else if let Some(back) = self.list.back {
    // We're on the ghost but non-empty, append to the back
    let in_front = input.front.take().unwrap();
    let in_back = input.back.take().unwrap();

    (*back.as_ptr()).back = Some(in_front);
    (*in_front.as_ptr()).front = Some(back);
    self.list.back = Some(in_back);
} else {
    // We're empty, become the input, remain on the ghost
    std::mem::swap(self.list, &mut input);
}

self.list.len += input.len;
// Not necessary but Polite To Do
input.len = 0;

// Input dropped here
```

Alright now this, this I can tolerate. The only complaints I have are that we no dedupe in_front/in_back (probably we could rejig our conditions but eh whatever). Really this is basically what you would write in C but with `Option<NonNull>` gunk making it tedious. I can live with that. Well no we should just make raw pointers better for this stuff. But, out of scope for this book.

En fin, después de esto estoy absolutamente agotado, así que `insert`, `remove` y las demás API se quedan como ejercicio para el lector.

Este es el código final de nuestro cursor, con mi intento de copiar y pegar las implementaciones combinatorias. ¿Lo he hecho bien? Solo lo descubriré cuando escriba el capítulo siguiente y pruebe esta monstruosidad.


```rust ,ignore
pub struct CursorMut<'a, T> {
    list: &'a mut LinkedList<T>,
    cur: Link<T>,
    index: Option<usize>,
}

impl<T> LinkedList<T> {
    pub fn cursor_mut(&mut self) -> CursorMut<T> {
        CursorMut { 
            list: self, 
            cur: None, 
            index: None,
        }
    }
}

impl<'a, T> CursorMut<'a, T> {
    pub fn index(&self) -> Option<usize> {
        self.index
    }

    pub fn move_next(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its next (back)
                self.cur = (*cur.as_ptr()).back;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() += 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real front, so move to it!
            self.cur = self.list.front;
            self.index = Some(0)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn move_prev(&mut self) {
        if let Some(cur) = self.cur {
            unsafe {
                // We're on a real element, go to its previous (front)
                self.cur = (*cur.as_ptr()).front;
                if self.cur.is_some() {
                    *self.index.as_mut().unwrap() -= 1;
                } else {
                    // We just walked to the ghost, no more index
                    self.index = None;
                }
            }
        } else if !self.list.is_empty() {
            // We're at the ghost, and there is a real back, so move to it!
            self.cur = self.list.back;
            self.index = Some(self.list.len - 1)
        } else {
            // We're at the ghost, but that's the only element... do nothing.
        }
    }

    pub fn current(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur.map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_next(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).back)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn peek_prev(&mut self) -> Option<&mut T> {
        unsafe {
            self.cur
                .and_then(|node| (*node.as_ptr()).front)
                .map(|node| &mut (*node.as_ptr()).elem)
        }
    }

    pub fn split_before(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                               ^
        //                              cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> C <-> D <- list.back
        //                   ^
        //                  cur
        //
        // 
        //    return.front -> A <-> B <- return.back
        //
        match self.index {
            // Edge case 1: We're at the ghost, just replace our list with an empty one.
            // No other state needs to be changed.
            None => std::mem::replace(self.list, LinkedList::new()),
            // Edge case 2: Cursor is at the first node: We don't have to do anything and hand out a new empty list.
            Some(0) => LinkedList::new(),
            // Base case: The list is non-empty and has at least 2 nodes.
            Some(old_idx) => unsafe {
                // Current state
                let old_len = self.list.len;
                let cur = self.cur.unwrap();
                let prev = (*cur.as_ptr()).front;

                // What self will become
                let new_len = old_len - old_idx;
                let new_front = self.cur;
                let new_back = self.list.back;
                let new_idx = Some(0);

                // What the output will become
                let output_len = old_idx;
                let output_front = self.list.front;
                let output_back = prev;

                // Break the links between cur and prev
                if let Some(prev) = prev {
                    (*cur.as_ptr()).front = None;
                    (*prev.as_ptr()).back = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            },
        }
    }

    pub fn split_after(&mut self) -> LinkedList<T> {
        // We have this:
        //
        //     list.front -> A <-> B <-> C <-> D <- list.back
        //                         ^
        //                        cur
        // 
        //
        // And we want to produce this:
        // 
        //     list.front -> A <-> B <- list.back
        //                         ^
        //                        cur
        //
        // 
        //    return.front -> C <-> D <- return.back
        //
        match self.index {
            None => std::mem::replace(self.list, LinkedList::new()),
            Some(idx) if idx + 1 == self.list.len => LinkedList::new(),
            // The list is non-empty and has at least 2 nodes.
            Some(old_idx) => unsafe {
                // Current state
                let old_len = self.list.len;
                let cur = self.cur.unwrap();
                let next = (*cur.as_ptr()).back;

                // What self will become
                let new_len = old_idx + 1;
                let new_back = self.cur;
                let new_front = self.list.front;
                let new_idx = Some(old_idx);

                // What the output will become
                let output_len = old_len - new_len;
                let output_front = next;
                let output_back = self.list.back;

                // Break the links between cur and next
                if let Some(next) = next {
                    (*cur.as_ptr()).back = None;
                    (*next.as_ptr()).front = None;
                }

                // Produce the result:
                self.list.len = new_len;
                self.list.front = new_front;
                self.list.back = new_back;
                self.index = new_idx;

                LinkedList {
                    front: output_front,
                    back: output_back,
                    len: output_len,
                    _boo: PhantomData,
                }
            },
        }
    }

    pub fn splice_before(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> 1 <-> 2 <-> B <-> C <- list.back
        //                                 ^
        //                                cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(prev) = (*cur.as_ptr()).front {
                    // General Case, no boundaries, just internal fixups
                    (*prev.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(prev);
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                } else {
                    // No prev, we're appending to the front
                    (*cur.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(cur);
                    self.list.front = Some(in_front);
                }
                // Index moves forward by input length
                *self.index.as_mut().unwrap() += input.len;
            } else if let Some(back) = self.list.back {
                // We're on the ghost but non-empty, append to the back
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*back.as_ptr()).back = Some(in_front);
                (*in_front.as_ptr()).front = Some(back);
                self.list.back = Some(in_back);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }

    pub fn splice_after(&mut self, mut input: LinkedList<T>) {
        // We have this:
        //
        // input.front -> 1 <-> 2 <- input.back
        //
        // list.front -> A <-> B <-> C <- list.back
        //                     ^
        //                    cur
        //
        //
        // Becoming this:
        //
        // list.front -> A <-> B <-> 1 <-> 2 <-> C <- list.back
        //                     ^
        //                    cur
        //
        unsafe {
            // We can either `take` the input's pointers or `mem::forget`
            // it. Using `take` is more responsible in case we ever do custom
            // allocators or something that also needs to be cleaned up!
            if input.is_empty() {
                // Input is empty, do nothing.
            } else if let Some(cur) = self.cur {
                // Both lists are non-empty
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                if let Some(next) = (*cur.as_ptr()).back {
                    // General Case, no boundaries, just internal fixups
                    (*next.as_ptr()).front = Some(in_back);
                    (*in_back.as_ptr()).back = Some(next);
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                } else {
                    // No next, we're appending to the back
                    (*cur.as_ptr()).back = Some(in_front);
                    (*in_front.as_ptr()).front = Some(cur);
                    self.list.back = Some(in_back);
                }
                // Index doesn't change
            } else if let Some(front) = self.list.front {
                // We're on the ghost but non-empty, append to the front
                let in_front = input.front.take().unwrap();
                let in_back = input.back.take().unwrap();

                (*front.as_ptr()).front = Some(in_back);
                (*in_back.as_ptr()).back = Some(front);
                self.list.front = Some(in_front);
            } else {
                // We're empty, become the input, remain on the ghost
                std::mem::swap(self.list, &mut input);
            }

            self.list.len += input.len;
            // Not necessary but Polite To Do
            input.len = 0;
            
            // Input dropped here
        }        
    }
}
```