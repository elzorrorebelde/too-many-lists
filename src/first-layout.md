# Diseño Básico de Datos

Muy bien, ¿qué es una lista enlazada? Bueno, básicamente es un montón de piezas de datos
en el heap (¡cállense, gente del kernel!) que se apuntan entre sí en secuencia. Las listas
enlazadas son algo con lo que los programadores procedurales no deberían ni acercarse,
y lo que los programadores funcionales usan para todo. Parece justo, entonces, que
preguntemos a los programadores funcionales por la definición de una lista enlazada. Probablemente
te darán algo como la siguiente definición:

```haskell
List a = Empty | Elem a (List a)
```

Que se lee aproximadamente como "Una Lista es o Vacía (Empty) o un Elemento seguido de una
Lista". Esta es una definición recursiva expresada como un *tipo suma (sum type)*, que es un
nombre elegante para "un tipo que puede tener diferentes valores que pueden ser de diferentes
tipos". ¡Rust llama a los tipos suma `enum`s! Si vienes de un lenguaje tipo C,
este es exactamente el enum que conoces y amas, pero en sobremarcha. Así que
transcribamos esta definición funcional a Rust.

Por ahora evitaremos los genéricos para mantener las cosas simples. Solo soportaremos
almacenar enteros signados de 32 bits:

```rust ,ignore
// in first.rs

// pub dice que queremos que la gente fuera de este módulo pueda usar List
pub enum List {
    Empty,
    Elem(i32, List),
}
```

*phew*, estoy agobiado. Compilemos eso:

```text
> cargo build

error[E0072]: recursive type `first::List` has infinite size
 --> src/first.rs:4:1
  |
4 | pub enum List {
  | ^^^^^^^^^^^^^ recursive type has infinite size
5 |     Empty,
6 |     Elem(i32, List),
  |               ---- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable
```

Bueno. No sé tú, pero yo ciertamente me siento traicionado por la comunidad de
programación funcional.

Si realmente revisamos el mensaje de error (después de superar todo el asunto de la
traición), podemos ver que rustc nos está diciendo exactamente
cómo resolver este problema:

> insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `first::List` representable

Muy bien, `box`. ¿Qué es eso? Busquemos `rust box`...

> [std::boxed::Box - Rust](https://doc.rust-lang.org/std/boxed/struct.Box.html)

Veamos...

> `pub struct Box<T>(_);`
>
> A pointer type for heap allocation.
> See the [module-level documentation](https://doc.rust-lang.org/std/boxed/) for more.

*hace clic en el enlace*

> `Box<T>`, casually referred to as a 'box', provides the simplest form of heap allocation in Rust. Boxes provide ownership for this allocation, and drop their contents when they go out of scope.
>
> Examples
>
> Creating a box:
>
> `let x = Box::new(5);`
>
> Creating a recursive data structure:
>
> ```rust
> #[derive(Debug)]
> enum List<T> {
>     Cons(T, Box<List<T>>),
>     Nil,
> }
> ```
>
> ```rust ,ignore
> fn main() {
>     let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
>     println!("{:?}", list);
> }
> ```
>
> This will print `Cons(1, Box(Cons(2, Box(Nil))))`.
>
> Recursive structures must be boxed, because if the definition of Cons looked like this:
>
> `Cons(T, List<T>),`
>
> It wouldn't work. This is because the size of a List depends on how many elements are in the list, and so we don't know how much memory to allocate for a Cons. By introducing a Box, which has a defined size, we know how big Cons needs to be.

Vaya, uh. Esa es quizás la documentación más relevante y útil que he visto nunca.
Literalmente lo primero en la documentación es *exactamente lo que estamos intentando
escribir, por qué no funcionó, y cómo solucionarlo*.

Caramba, la documentación mola.

Ok, hagamos eso:

```rust ,ignore
pub enum List {
    Empty,
    Elem(i32, Box<List>),
}
```

```text
> cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

¡Oye, compiló!

...pero esta es en realidad una definición realmente tonta de Lista, por varias razones.

Considera una lista con dos elementos:

```text
[] = Pila
() = Heap

[Elem A, ptr] -> (Elem B, ptr) -> (Empty, *basura*)
```

Hay dos problemas clave:

* Estamos asignando un nodo que solo dice "No soy realmente un Nodo"
* Uno de nuestros nodos no está asignado en el heap en absoluto.

En superficie, estos dos parecen cancelarse mutuamente. Asignamos en el heap un
nodo extra, pero uno de nuestros nodos no necesita estar en el heap.
Sin embargo, considera el siguiente diseño potencial para nuestra lista:

```text
[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
```

En este diseño ahora asignamos incondicionalmente nuestros nodos en el heap. La
diferencia clave es la ausencia de la *basura* de nuestro primer diseño. ¿Qué es
esta basura? Para entenderlo, necesitamos ver cómo se organiza un enum en la memoria.

En general, si tenemos un enum como:

```rust ,ignore
enum Foo {
    D1(T1),
    D2(T2),
    ...
    Dn(Tn),
}
```

Un Foo necesitará almacenar algún entero para indicar qué *variante* del enum
representa (`D1`, `D2`, .. `Dn`). Esta es la *etiqueta (tag)* del enum. También
necesitará suficiente espacio para almacenar el *más grande* de `T1`, `T2`, .. `Tn`
(más algo de espacio extra para satisfacer los requisitos de alineación).

La conclusión importante aquí es que aunque `Empty` es un solo bit de
información, necesariamente consume suficiente espacio para un puntero y un elemento,
porque tiene que estar preparado para convertirse en un `Elem` en cualquier momento.
Por lo tanto, el primer diseño asigna en el heap un elemento extra que está lleno de basura,
consumiendo un poco más de espacio que el segundo diseño.

Que uno de nuestros nodos no esté asignado en absoluto es también, quizás
sorprendentemente, *peor* que asignarlo siempre. Esto es porque nos da un diseño
de nodo *no uniforme*. Esto no tiene un efecto apreciable en push y pop de nodos,
pero sí tiene un efecto al dividir y fusionar listas.

Considera dividir una lista en ambos diseños:

```text
diseño 1:

[Elem A, ptr] -> (Elem B, ptr) -> (Elem C, ptr) -> (Empty *basura*)

dividir sacando C:

[Elem A, ptr] -> (Elem B, ptr) -> (Empty *basura*)
[Elem C, ptr] -> (Empty *basura*)
```

```text
diseño 2:

[ptr] -> (Elem A, ptr) -> (Elem B, ptr) -> (Elem C, *null*)

dividir sacando C:

[ptr] -> (Elem A, ptr) -> (Elem B, *null*)
[ptr] -> (Elem C, *null*)
```

La división del diseño 2 implica solo copiar el puntero de B a la pila y poner
el valor antiguo a null. El diseño 1 en última instancia hace lo mismo, pero también
tiene que copiar C del heap a la pila. Fusionar es el mismo proceso en reversa.

Una de las pocas cosas agradables de una lista enlazada es que puedes construir el
elemento en el propio nodo, y luego moverlo libremente entre listas sin
moverlo nunca. Solo manipulas punteros y las cosas se "mueven". El diseño 1
tira esta propiedad por la borda.

Muy bien, estoy razonablemente convencido de que el Diseño 1 es malo. ¿Cómo
reescribimos nuestra Lista? Bueno, podríamos hacer algo como:

```rust ,ignore
pub enum List {
    Empty,
    ElemThenEmpty(i32),
    ElemThenNotEmpty(i32, Box<List>),
}
```

Espero que esto te parezca una idea aún peor. Sobre todo, esto realmente
complica nuestra lógica, porque ahora hay un estado completamente inválido:
`ElemThenNotEmpty(0, Box(Empty))`. También *sigue* sufriendo de asignación no
uniforme de nuestros elementos.

Sin embargo, tiene *una* propiedad interesante: evita totalmente asignar
el caso Empty, reduciendo el número total de asignaciones en el heap en 1.
Desafortunadamente, al hacerlo logra *desperdiciar aún más espacio*. Esto es
porque el diseño anterior se aprovechaba de la *optimización de puntero nulo
(null pointer optimization)*.

Anteriormente vimos que cada enum tiene que almacenar una *etiqueta* para
especificar qué variante del enum representan sus bits. Sin embargo, si tenemos
un tipo especial de enum:

```rust,ignore
enum Foo {
    A,
    B(ContainsANonNullPtr),
}
```

la optimización de puntero nulo (null pointer optimization) entra en acción,
que *elimina el espacio necesario para la etiqueta*. Si la variante es A, todo
el enum se pone a `0`. De lo contrario, la variante es B. Esto funciona porque
B nunca puede ser todo `0`, ya que contiene un puntero no nulo. ¡Genial!

¿Se te ocurren otros enums y tipos que podrían hacer este tipo de optimización?
¡En realidad hay muchos! Por eso Rust deja el diseño de los enum totalmente sin
especificar. Hay algunas optimizaciones de diseño de enum más complicadas que
Rust hará por nosotros, ¡pero la del puntero nulo es definitivamente la más
importante! Significa que `&`, `&mut`, `Box`, `Rc`, `Arc`, `Vec`, y
varios otros tipos importantes en Rust no tienen sobrecarga cuando se ponen
en un `Option`. (Llegaremos a la mayoría de estos a su debido tiempo.)

Entonces, ¿cómo evitamos la basura extra, asignamos uniformemente, *y*
obtenemos esa dulce optimización de puntero nulo? Necesitamos separar mejor la
idea de tener un elemento de asignar otra lista. Para hacer esto, tenemos que
pensar un poco más como C: ¡con structs!

Mientras que los enums nos permiten declarar un tipo que puede contener *uno* de
varios valores, los structs nos permiten declarar un tipo que contiene *varios*
valores a la vez. Vamos a dividir nuestra Lista en dos tipos: una Lista y un Nodo.

Como antes, una Lista es o Vacía o tiene un elemento seguido de otra Lista.
Al representar el caso "tiene un elemento seguido de otra Lista" mediante un
tipo completamente separado, podemos elevar el Box a una posición más óptima:

```rust ,ignore
struct Node {
    elem: i32,
    next: List,
}

pub enum List {
    Empty,
    More(Box<Node>),
}
```

Revisemos nuestras prioridades:

* La cola de una lista nunca asigna basura extra: ¡correcto!
* El `enum` está en la deliciosa forma optimizada por puntero nulo: ¡correcto!
* Todos los elementos se asignan uniformemente: ¡correcto!

¡Muy bien! En realidad acabamos de construir exactamente el diseño que usamos
para demostrar que nuestro primer diseño (tal como lo sugería la documentación
oficial de Rust) era problemático.

```text
> cargo build

warning: private type `first::Node` in public interface (error E0446)
 --> src/first.rs:8:10
  |
8 |     More(Box<Node>),
  |          ^^^^^^^^^
  |
  = note: #[warn(private_in_public)] on by default
  = warning: this was previously accepted by the compiler but
    is being phased out; it will become a hard error in a future release!
```

:(

Rust está enfadado con nosotros otra vez. Marcamos el `List` como público (porque
queremos que la gente pueda usarlo), pero no el `Node`. El problema es que los
internos de un `enum` son totalmente públicos, y no se nos permite hablar
públicamente de tipos privados. Podríamos hacer que todo `Node` sea totalmente
público, pero generalmente en Rust preferimos mantener los detalles de
implementación privados. Envolvamos `List` dentro de un struct, para
poder ocultar los detalles de implementación:

```rust ,ignore
pub struct List {  // nuestro nuevo struct envoltorio
    head: Link,
}

enum Link {  // previamente conocido como List
    Empty,
    More(Box<Node>),
}

struct Node {
    elem: i32,
    next: Link,
}
```

Como `List` es un struct con un solo campo, su tamaño es el mismo que el de ese
campo. ¡Yupis, abstracciones de costo cero!

```text
> cargo build

warning: field is never used: `head`
 --> src/first.rs:2:5
  |
2 |     head: Link,
  |     ^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: variant is never constructed: `Empty`
 --> src/first.rs:6:5
  |
6 |     Empty,
  |     ^^^^^

warning: variant is never constructed: `More`
 --> src/first.rs:7:5
  |
7 |     More(Box<Node>),
  |     ^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/first.rs:11:5
   |
11 |     elem: i32,
   |     ^^^^^^^^^

warning: field is never used: `next`
  --> src/first.rs:12:5
   |
12 |     next: Link,
   |     ^^^^^^^^^^

```

¡Muy bien, compiló! Rust está bastante enfadado, porque por lo que puede ver,
todo lo que hemos escrito es totalmente inútil: nunca usamos `head`, y nadie que
use nuestra librería puede tampoco porque es privado. Transitivamente, eso
significa que Link y Node también son inútiles. ¡Así que solucionemos eso!
¡Implementemos algo de código para nuestra Lista!