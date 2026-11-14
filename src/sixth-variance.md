# Varianza y PhantomData

Sería molesto posponer esto y arreglarlo más tarde, así que vamos a ocuparnos ahora del diseño avanzado.

Hay cinco jinetes terribles al crear colecciones inseguras en Rust:

1. [Variance](https://doc.rust-lang.org/nightly/nomicon/subtyping.html)
2. [Drop Check](https://doc.rust-lang.org/nightly/nomicon/dropck.html)
3. [NonNull Optimizations](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)
4. [The isize::MAX Allocation Rule](https://doc.rust-lang.org/nightly/nomicon/vec/vec-alloc.html)
5. [Zero-Sized Types](https://doc.rust-lang.org/nightly/nomicon/vec/vec-zsts.html)

Por suerte, los dos últimos no serán un problema para nosotros.

El tercero *podríamos* convertirlo en un problema, pero daría más trabajo del que merece: si has elegido usar una `LinkedList`, ya has perdido por cien la batalla de la eficiencia de memoria.

El segundo es algo que antes insistía en considerar muy importante y con lo que `std` juega, pero los valores predeterminados son seguros, las formas de alterarlo son inestables y hay que esforzarse *muchísimo* para notar alguna limitación. Así que no te preocupes por ello.

Eso nos deja únicamente la varianza. Para ser sincero, probablemente también podríamos posponerla, pero conservo mi orgullo como persona dedicada a las colecciones, así que vamos a ocuparnos de la varianza.

Sorpresa: Rust tiene subtipado. En particular, `&'big T` es un *subtipo* de `&'small T`. ¿Por qué? Si un código necesita una referencia que viva durante una región concreta del programa, normalmente está bien darle una referencia que viva *más tiempo*. Intuitivamente es cierto, ¿verdad?

¿Por qué es importante? Imagina un código que recibe dos valores del mismo tipo:

```rust ,ignore
fn take_two<T>(_val1: T, _val2: T) { }
```

Este código es profundamente aburrido, así que esperamos que funcione bien con `T = &u32`, ¿verdad?

```rust
fn two_refs<'big: 'small, 'small>(
    big: &'big u32, 
    small: &'small u32,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

¡Sí, compila correctamente!

Ahora vamos a divertirnos y envolverlo en, no sé, `std::cell::Cell`:

```rust ,compilefail
use std::cell::Cell;

fn two_refs<'big: 'small, 'small>(
    // NOTE: these two lines changed
    big: Cell<&'big u32>, 
    small: Cell<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
error[E0623]: lifetime mismatch
 --> src/main.rs:7:19
  |
4 |     big: Cell<&'big u32>, 
  |               ---------
5 |     small: Cell<&'small u32>,
  |                 ----------- these two types are declared with different lifetimes...
6 | ) {
7 |     take_two(big, small);
  |                   ^^^^^ ...but data from `small` flows into `big` here
```

¿¡Eh!? No hemos tocado los tiempos de vida; ¿por qué se enfada ahora el compilador?

Bueno, el «subtipado» de tiempos de vida debe de ser muy sencillo, porque falla si envolvemos las referencias en cualquier cosa. Mira, también se rompe con `Vec`:

```rust
fn two_refs<'big: 'small, 'small>(
    big: Vec<&'big u32>, 
    small: Vec<&'small u32>,
) {
    take_two(big, small);
}

fn take_two<T>(_val1: T, _val2: T) { }
```

```text
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/playground`
```

Como ves, no compila ni siquie—espera, ¿¡qué!? ¿¡`Vec` es mágico!?

Bueno, sí. Pero también no. La magia siempre estuvo dentro de nosotros, y esa magia es ✨*la varianza*✨.

Lee el [capítulo del Nomicon sobre subtipado](https://doc.rust-lang.org/nightly/nomicon/subtyping.html) si quieres conocer todos los detalles escabrosos, pero, básicamente, el subtipado *no* siempre es seguro. En particular, no lo es cuando intervienen referencias mutables, porque puedes usar cosas como `mem::swap` y terminar de repente con punteros colgantes.

Las cosas que son «como referencias mutables» son *invariantes*, lo que significa que bloquean el subtipado de sus parámetros genéricos. Por seguridad, `&mut T` es invariante respecto de `T`, y `Cell<T>` es invariante respecto de `T` porque `&Cell<T>` es básicamente un `&mut T` debido a la mutabilidad interior.

Casi todo lo que no es invariante es *covariante*, lo que significa que el subtipado «atraviesa» el tipo y sigue funcionando normalmente. También existen tipos contravariantes que hacen que el subtipado funcione al revés, pero son muy raros y no le gustan a nadie, así que no volveré a mencionarlos.

Las colecciones suelen contener un puntero mutable a sus datos, así que podrías esperar que también fueran invariantes, pero en realidad no tienen por qué serlo. Gracias al sistema de propiedad de Rust, `Vec<T>` es semánticamente equivalente a `T`, por lo que es seguro que sea covariante.

Desafortunadamente, esta definición es invariante:

```rust
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = *mut Node<T>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

Pero ¿cómo decide Rust la varianza de las cosas? En los buenos tiempos, antes de la versión 1.0, probamos a dejar que cada persona especificara la varianza que quería y... ¡fue un desastre absoluto! El subtipado y la varianza son difíciles de entender, y los desarrolladores del núcleo ni siquiera se ponían de acuerdo en la terminología básica. Así que pasamos a un enfoque de «varianza por ejemplo»: el compilador mira los campos y copia sus varianzas. Si existe cualquier tipo de desacuerdo, siempre gana la invariancia, porque es la opción segura.

Entonces, ¿qué hay en nuestras definiciones de tipos que enfada a Rust? ¡`*mut`!

Los punteros crudos de Rust intentan dejarte hacer prácticamente cualquier cosa, pero tienen exactamente una característica de seguridad: como la mayoría de la gente no sabe que la varianza y el subtipado existen en Rust, y ser *incorrectamente* covariante sería muy peligroso, `*mut T` es invariante, porque es muy probable que se esté usando «como» un `&mut T`.

Esto resulta extremadamente molesto para Alguien Exactamente Como Yo, que ha pasado mucho tiempo escribiendo colecciones en Rust. Por eso, cuando creé [`std::ptr::NonNull`](https://doc.rust-lang.org/std/ptr/struct.NonNull.html), añadí este pequeño fragmento de magia:

> A diferencia de `*mut T`, se decidió que `NonNull<T>` fuera covariante respecto de `T`. Esto permite usar `NonNull<T>` al construir tipos covariantes, pero introduce el riesgo de que el código deje de ser correcto si se usa en un tipo que realmente no debería ser covariante.

Pero su interfaz está construida alrededor de `*mut T`; ¿qué está pasando? ¿Es magia? Veámoslo:

```rust
pub struct NonNull<T> {
    pointer: *const T,
}


impl<T> NonNull<T> {
    pub unsafe fn new_unchecked(ptr: *mut T) -> Self {
        // SAFETY: the caller must guarantee that `ptr` is non-null.
        unsafe { NonNull { pointer: ptr as *const T } }
    }
}
```

¡NO! ¡AQUÍ NO HAY MAGIA! `NonNull` se aprovecha de que `*const T` es covariante y almacena eso en su lugar, convirtiendo entre `*mut T` en el límite de la API para que «parezca» que almacena un `*mut T`. ¡Ese es todo el truco! ¡Así es como las colecciones de Rust son covariantes! ¡Y es miserable! Por eso hice que el tipo de puntero correcto lo hiciera por ti. ¡De nada! ¡Disfruta de tu peligrosa trampa de subtipado!

La solución a todos tus problemas es usar `NonNull` y, si quieres volver a tener punteros anulables, usar `Option<NonNull<T>>`. ¿De verdad vamos a molestarnos en hacerlo?

¡Sí! Es un fastidio, pero estamos creando *listas enlazadas de calidad de producción*, así que nos comeremos todas las verduras y haremos las cosas por el camino difícil. Podríamos usar `*const T` sin más y convertirlo en todas partes, pero de verdad quiero comprobar cuánto duele esto... por la ciencia de la ergonomía.


Estas son nuestras definiciones de tipos finales:

```rust
use std::ptr::NonNull;

// !!!This changed!!!
pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
}

type Link<T> = Option<NonNull<Node<T>>>;

struct Node<T> {
    front: Link<T>,
    back: Link<T>,
    elem: T, 
}
```

...espera, no: una última cosa. Cada vez que trabajes con punteros crudos deberías añadir un fantasma para protegerlos:

```rust ,ignore
use std::marker::PhantomData;

pub struct LinkedList<T> {
    front: Link<T>,
    back: Link<T>,
    len: usize,
    /// We semantically store values of T by-value.
    _boo: PhantomData<T>,
}
```

En este caso no creo que *realmente* necesitemos [`PhantomData`](https://doc.rust-lang.org/std/marker/struct.PhantomData.html), pero cada vez que uses `NonNull` (o punteros crudos en general) deberías añadirlo para estar seguro y dejar claro al compilador y a los demás lo que *crees* que estás haciendo.

`PhantomData` nos permite dar al compilador un campo «de ejemplo» adicional que *conceptualmente* existe en nuestro tipo, pero que por diversos motivos (indirección, borrado de tipos, ...) no existe realmente. En este caso usamos `NonNull` porque afirmamos que nuestro tipo se comporta «como si» almacenara un valor `T`, así que añadimos `PhantomData` para hacerlo explícito.

La biblioteca estándar tiene otros motivos para hacerlo, porque dispone de los malditos [anuladores de comprobación de `Drop`](https://doc.rust-lang.org/nightly/nomicon/dropck.html), pero esa funcionalidad se ha rediseñado tantas veces que ya no sé si lo de `PhantomData` sigue siendo necesario. Aun así, seguiré copiándolo por inercia eternamente, porque la magia de `Drop Check` está grabada en mi cerebro.

(El `Node` almacena literalmente un `T`, así que no tiene que hacer esto. ¡Bien!)

...de acuerdo, ahora sí hemos terminado con el diseño. ¡Pasemos a la funcionalidad básica!
