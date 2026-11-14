# Probando Stacked Borrows

> Resumen del modelo de memoria (simplificado) de Rust de la sección anterior:
>
> * Rust gestiona conceptualmente los nuevos préstamos manteniendo una «pila de préstamos».
> * Solo el préstamo que está en la parte superior está «activo» (tiene acceso exclusivo).
> * Cuando accedes a uno inferior, este pasa a estar «activo» y los que están por encima se extraen de la pila.
> * No se permite usar punteros que se hayan extraído de la pila de préstamos.
> * El comprobador de préstamos garantiza que el código seguro cumpla estas reglas.
> * En teoría, `miri` comprueba en tiempo de ejecución que los punteros crudos las cumplan.

Ha sido mucha teoría e ideas; pasemos al verdadero corazón de este libro: escribir código incorrecto y hacer que nuestras herramientas nos griten. Recorreremos un *montón* de ejemplos para comprobar si nuestro modelo mental tiene sentido y adquirir una intuición sobre los préstamos apilados.

> **NARRADOR:** Detectar comportamiento indefinido en la práctica es complicado. Al fin y al cabo, estás tratando con situaciones que el compilador literalmente *supone* que no ocurren.
>
> Si tienes suerte, hoy las cosas «parecerán funcionar», pero serán una bomba de relojería para un compilador más inteligente o para un pequeño cambio en el código. Si tienes *mucha* suerte, fallarán de forma reproducible, para que puedas detectar y arreglar el error. Pero si no tienes suerte, todo se romperá de formas extrañas y desconcertantes.
>
> `Miri` intenta solucionar esto obteniendo la visión más ingenua y sin optimizar del programa que tiene `rustc` y siguiendo estado adicional mientras lo interpreta. En cuanto a «sanitizadores» se refiere, es un enfoque bastante determinista y robusto, pero nunca será *perfecto*. El programa de prueba debe ejecutar realmente ese comportamiento indefinido y, en un programa suficientemente grande, es muy fácil introducir todo tipo de no determinismo (¡los `HashMap` usan un generador aleatorio de forma predeterminada!).
>
> Nunca podemos interpretar que la aprobación de `miri` sea una afirmación absolutamente cierta de que no hay comportamiento indefinido. También es posible que `miri` *crea* que algo es comportamiento indefinido cuando en realidad no lo es. Pero si tenemos un modelo mental de cómo funciona todo y `miri` parece estar de acuerdo, es una buena señal de que vamos por el camino correcto.




# Basic Borrows

En las secciones anteriores vimos que al comprobador de préstamos no le gusta este código:

```rust ,ignore
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// ORDER SWAPPED!
*ref1 += 1;
*ref2 += 2;

println!("{}", data);
```

Veamos qué ocurre cuando sustituimos `ref2` por `*mut`:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;

    // ORDER SWAPPED!
    *ref1 += 1;
    *ptr2 += 2;

    println!("{}", data);
}
```

```text
cargo run
   Compiling miri-sandbox v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 0.71s
     Running `target\debug\miri-sandbox.exe`
13
```

`rustc` parece completamente satisfecho: no hay advertencias y el programa produjo el resultado esperado. Veamos ahora qué opina `miri` (en modo estricto):

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running cargo-miri.exe target\miri

error: Undefined Behavior: no item granting read access 
to tag <untagged> at alloc748 found in borrow stack.

 --> src\main.rs:9:9
  |
9 |         *ptr2 += 2;
  |         ^^^^^^^^^^ no item granting read access to tag <untagged> 
  |                    at alloc748 found in borrow stack.
  |
  = help: this indicates a potential bug in the program: 
    it performed an invalid operation, but the rules it 
    violated are still experimental
 
```

¡Bien! Nuestro modelo intuitivo de cómo funciona todo se mantuvo: aunque el compilador no pudo detectar el problema, `miri` sí pudo.

Probemos algo más complicado: el caso `&mut -> *mut -> &mut -> *mut` que mencionamos antes:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // Access the first raw pointer first
    *ptr2 += 2;

    // Then access things in "borrow stack" order
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
22

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1621> at alloc748 found in borrow stack.

  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting read access to tag <1621> 
   |                at alloc748 found in borrow stack.
   |
```

¡Vaya, sí! En modo estricto, `miri` puede «distinguir» los dos punteros crudos y hacer que usar el segundo invalide el primero. Veamos si todo funciona cuando eliminamos el primer uso, que es el que lo estropea todo:

```rust ,ignore
unsafe {
    let mut data = 10;
    let ref1 = &mut data;
    let ptr2 = ref1 as *mut _;
    let ref3 = &mut *ptr2;
    let ptr4 = ref3 as *mut _;

    // Access things in "borrow stack" order
    *ptr4 += 4;
    *ref3 += 3;
    *ptr2 += 2;
    *ref1 += 1;

    println!("{}", data);
}
```

```text
cargo run
20

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
20
```

NICE.

A estas alturas estoy bastante seguro de que todos podemos conseguir un doctorado en diseño e implementación de modelos de memoria de lenguajes de programación. ¿Quién necesita compiladores? ¡Esto es *facilísimo*!

> **NARRATOR:** it was not, but Estoy proud of you nonetheless.





# Testing Arrays

Juguemos con algunos arrays y desplazamientos de punteros (`add` y `sub`). Esto debería funcionar, ¿verdad?

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr3_at_1 = ptr2_at_0.add(1);       // Ptr to 1st element

    *ptr3_at_1 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[3, 3, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1619> at alloc748+0x4 found in borrow stack.
 --> src\main.rs:8:5
  |
8 |     *ptr3_at_1 += 3;
  |     ^^^^^^^^^^^^^^^ no item granting read access to tag <1619>
  |                     at alloc748+0x4 found in borrow stack.
```

*Rips up gradschool application*

¿Qué ha ocurrido? ¡Estamos usando perfectamente la pila de préstamos! ¿Sucede algo extraño cuando pasamos de `ptr -> ptr`? ¿Y si simplemente copiamos el puntero para que todos apunten al mismo lugar?

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;              // Ptr to 0th element

    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [6, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[6, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

No, eso funciona bien. Quizá estemos teniendo suerte; hagamos un auténtico lío de punteros:

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];            // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;   // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;               // Ptr to 0th element
    let ptr4_at_0 = ptr2_at_0.add(0);        // Ptr to 0th element
    let ptr5_at_0 = ptr3_at_0.add(1).sub(1); // Ptr to 0th element

    // An absolute jumbled hash of ptr usages
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ptr4_at_0 += 4;
    *ptr5_at_0 += 5;
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [20, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[20, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

¡No! `Miri` es mucho más permisivo con los punteros crudos derivados de otros punteros crudos. Todos comparten el mismo «préstamo» (o *etiqueta*, como lo llama `miri`).

Una vez que empiezas a usar punteros crudos, estos pueden dividirse libremente en sus propios hombrecillos enfadados y jugar entre sí. No pasa nada, porque el compilador entiende esto y no optimiza las lecturas y escrituras del mismo modo que lo hace con las referencias.

> **NARRADOR:** Si el código es lo bastante sencillo, el compilador puede seguir todos los punteros derivados y optimizar lo que sea posible, pero este razonamiento será mucho más frágil que el que puede aplicar a las referencias.

So what's the *real* problem?

Aunque `data` es una única «asignación» (una variable local), `ref1_at_0` solo toma prestado el primer elemento. Rust permite dividir los préstamos para que solo se apliquen a partes concretas de la asignación. Probémoslo:

```rust ,ignore
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ref2_at_1 = &mut data[1];           // Reference to 1th element
    let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr4_at_1 = ref2_at_1 as *mut i32;   // Ptr to 1th element

    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;
    *ref2_at_1 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
error[E0499]: cannot borrow `data[_]` as mutable more than once at a time
 --> src\main.rs:5:21
  |
4 |     let ref1_at_0 = &mut data[0];           // Reference to 0th element
  |                     ------------ first mutable borrow occurs here
5 |     let ref2_at_1 = &mut data[1];           // Reference to 1th element
  |                     ^^^^^^^^^^^^ second mutable borrow occurs here
6 |     let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
  |                     --------- first borrow later used here
  |
  = help: consider using `.split_at_mut(position)` or similar method 
    to obtain two mutable non-overlapping sub-slices
```

¡Rayos! Rust no sigue los índices de los arrays para demostrar que estos préstamos son disjuntos, pero nos proporciona `split_at_mut` para dividir un slice en varias partes de una forma que es seguro asumir que funciona:

```rust
unsafe {
    let mut data = [0; 10];

    let slice1 = &mut data[..];
    let (slice2_at_0, slice3_at_1) = slice1.split_at_mut(1); 
    
    let ref4_at_0 = &mut slice2_at_0[0];    // Reference to 0th element
    let ref5_at_1 = &mut slice3_at_1[0];    // Reference to 1th element
    let ptr6_at_0 = ref4_at_0 as *mut i32;  // Ptr to 0th element
    let ptr7_at_1 = ref5_at_1 as *mut i32;  // Ptr to 1th element

    *ptr7_at_1 += 7;
    *ptr6_at_0 += 6;
    *ref5_at_1 += 5;
    *ref4_at_0 += 4;

    // Should be [10, 12, 0, ...]
    println!("{:?}", &data[..]);
}
```

```text
cargo run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[10, 12, 0, 0, 0, 0, 0, 0, 0, 0]
```

¡Oye, funciona! Los slices indican correctamente al compilador y a `miri`: «estoy tomando prestada toda la memoria de mi rango», así que saben que todos los elementos pueden modificarse.

Observa también que permitir operaciones como `split_at_mut` indica que los préstamos pueden parecer menos una *pila* y más un *árbol*: podemos dividir un préstamo grande en varios préstamos pequeños y disjuntos, y todo sigue funcionando.

(Creo que en el modelo real de préstamos apilados todo sigue siendo una pila, porque conceptualmente las pilas siguen los permisos de cada byte del programa... ¿no?)

What if we *directly* turn a slice into a pointer? Will that pointer have access to the full slice?

```rust
unsafe {
    let mut data = [0; 10];

    let slice1_all = &mut data[..];         // Slice for the entire array
    let ptr2_all = slice1_all.as_mut_ptr(); // Pointer for the entire array
    
    let ptr3_at_0 = ptr2_all;               // Pointer to 0th elem (the same)
    let ptr4_at_1 = ptr2_all.add(1);        // Pointer to 1th elem
    let ref5_at_0 = &mut *ptr3_at_0;        // Reference to 0th elem
    let ref6_at_1 = &mut *ptr4_at_1;        // Reference to 1th elem

    *ref6_at_1 += 6;
    *ref5_at_0 += 5;
    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;

    // Just for fun, modify all the elements in a loop
    // (Could use any of the raw pointers for this, they share a borrow!)
    for idx in 0..10 {
        *ptr2_all.add(idx) += idx;
    }

    // Safe version of this same code for fun
    for (idx, elem_ref) in slice1_all.iter_mut().enumerate() {
        *elem_ref += idx; 
    }

    // Should be [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
    println!("{:?}", &data[..]);
}
```


```text
cargo run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
[8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
```


¡Genial! Los punteros no son simples enteros: tienen asociado un rango de memoria y Rust nos permite estrechar ese rango.





# Testing Shared References

En todos estos ejemplos he usado cuidadosamente solo referencias mutables y operaciones de lectura-modificación-escritura (`+=`) para mantener las cosas lo más sencillas posible.

Pero Rust también tiene referencias compartidas, que son de solo lectura y se pueden copiar libremente. ¿Cómo deberían funcionar? Hemos visto que los punteros crudos se pueden copiar libremente y que podemos tratarlo diciendo que «comparten» un único préstamo. Quizá podamos pensar lo mismo de las referencias compartidas.

Probémoslo con una función que lea un valor (`println!` puede hacer magia con las conversiones automáticas de referencias y desreferencias, así que la envolveré en una función para asegurarnos de probar lo que queremos):

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let sref2 = &mref1;
    let sref3 = sref2;
    let sref4 = &*sref2;

    // Random hash of shared reference reads
    opaque_read(sref3);
    opaque_read(sref2);
    opaque_read(sref4);
    opaque_read(sref2);
    opaque_read(sref3);

    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

warning: unnecessary `unsafe` block
 --> src\main.rs:6:1
  |
6 | unsafe {
  | ^^^^^^ unnecessary `unsafe` block
  |
  = note: `#[warn(unused_unsafe)]` on by default

warning: `miri-sandbox` (bin "miri-sandbox") generated 1 warning

10
10
10
10
10
11
```

Ah, sí, hemos olvidado hacer nada con punteros crudos, pero al menos podemos ver que está bien usar indistintamente todas las referencias compartidas. Mezclemos ahora algunos punteros crudos:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &mref1;
    let ptr4 = sref3 as *mut i32;

    *ptr4 += 4;
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run

error[E0606]: casting `&&mut i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

Vaya, en realidad estábamos trabajando con `& &mut` en lugar de `&`. Rust es muy bueno ocultando esa diferencia cuando no importa. Volvamos a tomarlo prestado correctamente con `let sref3 = &*mref1`:


```text
cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src\main.rs:11:16
   |
11 |     let ptr4 = sref3 as *mut i32;
   |                ^^^^^^^^^^^^^^^^^
```

No, a Rust todavía no le gusta. Una referencia compartida solo se puede convertir en un `*const`, que únicamente puede leer. Pero ¿y si simplemente... hacemos... esto...?

```rust ,ignore
    let ptr4 = sref3 as *const i32 as *mut i32;
```

```text
cargo run

14
17
```

¿¡QUÉ!? DE ACUERDO, ¿¡VALE!? Menudo sistema de conversiones, Rust. Es casi como si `*const` fuera un tipo bastante inútil que solo existe para describir API de C y sugerir vagamente el uso correcto (y así es, en efecto). ¿Qué opina `miri`?

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting write access to 
tag <1621> at alloc742 found in borrow stack.
  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting write access to tag <1621>
   |                at alloc742 found in borrow stack.
```

Aunque podemos evitar la queja del compilador con una conversión doble, eso no hace que la operación esté *permitida*. Al tomar la referencia compartida, prometemos no modificar el valor.

Esto es importante porque significa que, cuando el préstamo compartido se extrae de la pila, los punteros mutables que están debajo *pueden* asumir que la memoria no ha cambiado. Puede que algunos hombrecillos enfadados hayan *leído* la memoria (por lo que hubo que confirmar las escrituras), pero no pudieron modificarla y los punteros mutables pueden asumir que su último valor escrito sigue allí.

**Una vez que hay una referencia compartida en la pila de préstamos, todo lo que se coloque encima solo tiene permisos de lectura.**

We can however do this:

```rust
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;
    let ptr4 = sref3 as *const i32 as *mut i32;

    opaque_read(&*ptr4);
    opaque_read(sref3);
    *ptr2 += 2;
    *mref1 += 1;

    opaque_read(&data);
}
```

Observa que seguía estando «bien» crear un puntero crudo mutable siempre que solo leyéramos a través de él.

```text
cargo run
10
10
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
10
10
13
```

And just to be sure, let's check that a shared reference gets popped like normal:

```rust ,ignore
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;

    *ptr2 += 2;
    opaque_read(sref3); // Read in the wrong order?
    *mref1 += 1;

    opaque_read(&data);
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly 
at alloc742, but parent tag <1620> does not have an appropriate 
item in the borrow stack

  --> src\main.rs:13:17
   |
13 |     opaque_read(sref3); // Read in the wrong order?
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc742, but parent tag <1620> 
   |                       does not have an appropriate item 
   |                       in the borrow stack
   |
```

Hey, we even got a slightly different error message about SharedReadOnly instead of some specific tag. That makes sense: once hay *any* shared references, basically everything else is just a big SharedReadOnly soup so hay no need to distinguish any of them!





# Testing Interior Mutability

Remember that really horrible chapter of the book where we tried to make a linked list with RefCell and Rc and everything was even worse than usual when trying to write this godforsaken linked lists?

Hemos been insisting shared references no puede be used for mutation but that chapter was all about how you could actually mutate through shared references with *interior mutability*. Vamos a try the nice and simple [std::cell::Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) type:

```rust
use std::cell::Cell;

unsafe {
    let mut data = Cell::new(10);
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut Cell<i32>;
    let sref3 = &*mref1;

    sref3.set(sref3.get() + 3);
    (*ptr2).set((*ptr2).get() + 2);
    mref1.set(mref1.get() + 1);

    println!("{}", data.get());
}
```

Ah, such a beautiful mess. It will be lovely to see miri spit on it.


```text
cargo run
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
16
```

Wait, really? *That's* fine? Why? How? What even is a *Cell*?

*Smashes the padlock on the stdlib*

```rust ,ignore
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

What the heck is `UnsafeCell`?

*Smashes another padlock just to really show the stdlib we mean business*

```rust ,ignore
#[lang = "unsafe_cell"]
#[repr(transparent)]
#[repr(no_niche)]
pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
```

Oh it's wizard magic. Ok. I guess. `#[lang = "unsafe_cell"]` is literally just saying UnsafeCell is UnsafeCell. Vamos a stop breaking locks and check the actual documentation of [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html).

> The core primitive for interior mutability in Rust.
>
> If you have a reference `&T`, then normally in Rust the compiler performs optimizations based on the knowledge that `&T` points to immutable data. Mutating that data, for example through an alias or by transmuting an `&T` into an `&mut T`, is considered undefined behavior. `UnsafeCell<T>` opts-out of the immutability guarantee for `&T`: a shared reference `&UnsafeCell<T>` may point to data that is being mutated. This is called “interior mutability”.

Oh it *really is* just wizard magic.

UnsafeCell basically tells the compiler "hey listen, we're gonna get goofy with this memory, no make any of the usual aliasing assumptions about it". Like putting up a big "CAUTION: TINY ANGRY MEN CROSSING" sign.

Vamos a see how adding UnsafeCell makes miri happy:

```rust ,ignore
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = data.get_mut();      // Get a mutable ref to the contents
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*ptr2;

    *ptr2 += 2;
    opaque_read(sref3);
    *mref1 += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly
at alloc748, but parent tag <1629> does not have an appropriate
item in the borrow stack

  --> src\main.rs:15:17
   |
15 |     opaque_read(sref3);
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc748, but parent tag <1629> does
   |                       not have an appropriate item in the
   |                       borrow stack
   |

```

Wait, what? We spoke the magic words! What am I going to do with all this federally approved ritual-enhancing goat blood?

Well, we did, but then we completely discarded the spell by using `get_mut` which peeks inside the UnsafeCell and makes a proper `&mut i32` to it anyway!

Think about it: if the compiler had to assume `&mut i32` *could* be looking inside an `UnsafeCell`, then it would never be able to make any assumptions about aliasing at all! Everything could be full of tiny angry men.

So what we need to do is keep the `UnsafeCell` in our pointer types so that the compiler understands what we're doing.

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;              // Mutable ref to the *outside*
    let ptr2 = mref1.get();             // Get a raw pointer to the insides
    let sref3 = &*mref1;                // Get a shared ref to the *outside*

    *ptr2 += 2;                         // Mutate with the raw pointer
    opaque_read(&*sref3.get());         // Read from the shared ref
    *sref3.get() += 3;                  // Write through the shared ref
    *mref1.get() += 1;                  // Mutate with the mutable ref

    println!("{}", *data.get());
}
```


```text
cargo run
12
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
12
16
```

It works! I no have to throw out all this blood after all.

Actually, hey wait. Estamos still being a bit goofy with the order here. We made ptr2 first, and then made sref3 from the mutable pointer. And then we used the raw pointer before the shared pointer. That all seems... wrong.

Actually wait we did that with the Cell example too. HMMM.

Estamos forced to conclude one of two things:

* Miri is imperfect and this is actually still UB.
* Our simplified model is in fact an oversimplication.

I'd put my money on the second one, but just to be safe let's make a version that's definitely airtight in our simplified model of stacked borrows:

```rust
use std::cell::UnsafeCell;

fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = UnsafeCell::new(10);
    let mref1 = &mut data;
    // These two are swapped so the borrows are *definitely* totally stacked
    let sref2 = &*mref1;
    // Derive the ptr from the shared ref to be super safe!
    let ptr3 = sref2.get();             

    *ptr3 += 3;
    opaque_read(&*sref2.get());
    *sref2.get() += 2;
    *mref1.get() += 1;

    println!("{}", *data.get());
}
```

```text
cargo run
13
16

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
13
16
```

Now, one reason why the first implementation we had *might* actually be correct is because if you *really* think about it `&UnsafeCell<T>` really is no different from `*mut T` as far as aliasing is concerned. You can infinitely copy it and mutate through it!

So in some sense we just created two raw pointers and used them interchangeably like normal. Es *a little* sketchy that both were derived from the mutable reference, so maybe the second one's creation should still pop the first one off the borrow stack, but that's not really necessary since we're not *actually* accessing the contents of the mutable reference, just copying its address.

A line like `let sref2 = &*mref1` is a tricksy thing. *Syntactically* it looks like we're dereferencing it, but dereferencing on it's own no es actually a *thing*? Consider `&my_tuple.0`: you no son actually doing anything to `my_tuple` or `.0`, you're just using them to refer to a location in memory and putting `&` in front of it that says "no load this, just write the address down". 

`&*` is the same thing: the `*` is just saying "hey let's talk about the location this pointer points to" and the `&` is just saying "now write that address down". Which is of course the same value the original pointer had. But the type of the pointer has changed, because, uh, types!

That said, if you do `&**` then you are in fact loading a value with the first `*`! `*` is weird!

> **NARRATOR:** No one cares that you know the word "lvalue", *Jonathan*. In Rust we call them *places*, which is totally different and *so* much cooler?




# Testing Box

Hey remember why we started this extremely long aside? You don't? Weird.

Well it was because we mixed Box and raw pointers. Box is *kind of* like `&mut`, because it claims unique ownership of the memory it points to. Vamos a test that claim out:

```rust ,ignore
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1;

    // Should be 21
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
       to tag <1707> at alloc763 found in borrow stack.

 --> src\main.rs:7:5
  |
7 |     *ptr1 += 1;
  |     ^^^^^^^^^^ no item granting read access to tag <1707> 
  |                at alloc763 found in borrow stack.
  |
```

Yep, miri hates that. Vamos a check that doing things in the right order is ok:

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *ptr1 += 1;
    *data += 10;

    // Should be 21
    println!("{}", data);
}
```

```text
cargo run
21

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run
21
```

Yep!

Whelp that's all folks, we're finally done talking and thinking about stacked borrows!

...wait how do we solve this problem with Box? Like, sure we can write toy programs like this but we need to store the Box somewhere and hold onto our raw pointers for a potentially long time. Surely stuff will get mixed up and invalidated?

Great question! To answer that we'll finally be returning to our true calling: writing some god damn linked lists.

Wait, I need to write linked lists again? Vamos a not be hasty folks. Be reasonable. Just hold on Estoy sure hay some other interesting issues for me to discu&mdash;