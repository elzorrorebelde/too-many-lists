# Intentando Entender Stacked Borrows

En la sección anterior intentamos ejecutar nuestra cola enlazada simple insegura bajo `miri`, y nos indicó que habíamos infringido las reglas de los *préstamos apilados* y nos enlazó documentación.

Normalmente haría un recorrido guiado por la documentación, pero no somos realmente su público objetivo. Está más bien pensada para desarrolladores del compilador y académicos que trabajan en la semántica de Rust.

Así que solo voy a explicar la *idea* general de los «préstamos apilados» y después ofreceré una estrategia sencilla para seguir las reglas.

> **NARRADOR:** Los préstamos apilados siguen siendo «experimentales» como modelo semántico de Rust, así que infringir estas reglas no significa necesariamente que tu programa esté «mal». Pero, a menos que trabajes literalmente en el compilador, deberías arreglar el programa cuando `miri` se queje. Más vale prevenir que lamentar cuando se trata de comportamiento indefinido.



# La motivación: aliasing de punteros

Antes de explicar *qué* reglas hemos infringido, será útil entender *por qué* existen. Hay varios problemas que las motivan, pero creo que el más importante es el *aliasing de punteros*.

Decimos que dos punteros tienen *aliasing* cuando se solapan las zonas de memoria a las que apuntan. Igual que alguien que «usa un alias» puede ser llamado por dos nombres distintos, esa zona de memoria solapada puede estar referenciada por dos punteros diferentes. Esto puede causar problemas.

El compilador utiliza la información sobre el aliasing de punteros para optimizar los accesos a memoria, así que si esa información es *incorrecta*, el programa se compilará mal y hará cosas absurdas.

> **NARRADOR:** En la práctica, el aliasing tiene más que ver con los accesos a memoria que con los punteros en sí, y solo importa realmente cuando uno de los accesos es mutable. Se hace hincapié en los punteros porque son un elemento cómodo al que asociar reglas.

Para entender por qué es importante la información sobre el aliasing de punteros, consideremos *la parábola del hombrecillo enfadado*.

----

Un día, Michiel estaba mirando su estantería cuando vio un libro que no recordaba. Lo sacó de la estantería y miró la portada.

«Ah, sí, mi viejo ejemplar de *Guerra y paz*, un libro que sin duda he leído. Me encantó la parte de toda la paz.»

Suddenly there was a knock at the door. Michiel returned the book to its shelf and opened the door -- it was their sworn nemesis **Hamslaw**. As Hamslaw prepared a devastating remark about Michiel's clearly inferior codegolfing skills, they sensed an opening: 

"Hey Hamslaw, have you ever read War and Peace?"

"Pfft, no one's *actually* read War and Peace."

"Well I have, look it's right there in my bookcase, which *obviously* means He read it."

Hamslaw no podía creérselo. Su rostro pasó de la habitual suficiencia a una máscara de hierro llena de rabia y determinación. Apartó a Michiel y caminó decididamente hasta la estantería, arrancando el tomo de su sitio con la furia de mil valquirias. Dio la vuelta al texto antiguo entre sus manos y, en cuanto vio la portada, empezó a temblar.

Michiel prepared to boast of their clearly unparalleled brilliance, but was interrupted by the sudden laughter of Hamslaw.

«¡Esto no es *Guerra y paz*, es *Guerra y pies*!»

Tears were rolling down Hamslaw's face. This was clearly the greatest moment of her life.

"N-no! I just looked at it!"

They grabbed the book from Hamslaw and checked the cover. Indeed, the word "Peace" had been scratched out and replaced with "Feet". Michiel was mortified. This was clearly the worst moment of their life.

They fell to their knees and stared blankly at the bookcase. How could this have happened? They had checked the cover only a moment ago!

And then they saw a bit of motion in the bookcase. It was a tiny man. A tiny man with the angriest scowl Michiel had ever seen. The tiny man flipped Michiel off and mouthed the words "no one will believe you" and disappeared back between the books.

Michiel's plan *had* been perfect, but they had failed to account for the possibility of a tiny angry man with a sharpie and the desire for destruction. They thought they knew what the cover of the book said, and they thought that no one could have possibly changed it. But alas, they were wrong.

Hamslaw was already working on a zine commemorating her incredible victory &mdash; Michiel's reputation at the local Internet Cafe would never recover.

----

No one wants to be like Michiel, but no one wants to live in constant fear of the tiny angry man either. We want to know when the tiny angry man could be playing tricks on us. When he is, we will be very careful and paranoid about checking everything before we use it. But when the tiny angry man is gone, we want to be able to remember things.

Ese es el meollo (muy simplificado) del aliasing de punteros: ¿cuándo puede el compilador suponer que es seguro «recordar» (guardar en caché) valores en lugar de cargarlos una y otra vez? Para saberlo, el compilador necesita determinar si *podría* haber hombrecillos enfadados modificando la memoria a tus espaldas.

> **NARRATOR:** the compiler also uses this information to cache stores, which just means it can avoid committing things to memory if it thinks no one will notice. In this case the problem is still tiny angry men, but they only need to read the memory for it to be a problem.






# Préstamos apilados seguros

Queremos que el compilador tenga buena información sobre el aliasing de punteros. ¿Podemos conseguirlo? Parece que Rust está *diseñado* para ello. Por definición, las referencias mutables no tienen alias, y aunque las referencias compartidas *pueden* tener alias, no pueden modificar datos. ¡Perfecto, enviémoslo!

Except it's more complicated than that. We can "reborrow" mutable pointers like this:

```rust
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

*ref2 += 2;
*ref1 += 1;

println!("{}", data);
```

Compila y se ejecuta correctamente. ¿Qué ocurre?

Well we can see what's going on by swapping the two uses:

```rust ,ignore
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// ORDER SWAPPED!
*ref1 += 1;
*ref2 += 2;

println!("{}", data);
```

```text
error[E0503]: cannot use `*ref1` because it was mutably borrowed
 --> src/main.rs:6:5
  |
4 |     let ref2 = &mut *ref1;
  |                ---------- borrow of `*ref1` occurs here
5 |     
6 |     *ref1 += 1;
  |     ^^^^^^^^^^ use of borrowed `*ref1`
7 |     *ref2 += 2;
  |     ---------- borrow later used here

For more information about this error, try `rustc --explain E0503`.
error: could not compile `playground` due to previous error
```

¡De repente es un error del compilador!

Cuando volvemos a tomar prestado un puntero mutable, ya no podemos usar el puntero original hasta que el nuevo préstamo haya terminado (es decir, hasta que no haya más usos).

En el código que funciona, los usos están bien anidados. Volvemos a tomar prestado el puntero, usamos el nuevo durante un tiempo y dejamos de usarlo antes de volver a usar el puntero antiguo. En el código que *no* funciona, eso no ocurre: intercalamos los usos arbitrariamente.

This is how we can have reborrows and still have aliasing information: all of our reborrows clearly nest, so we can consider only one of them "live" at any given time.

Hey, you know what's a great way to represent cleanly nested things? A stack. A stack of borrows.

Oh hey it's *Stacked Borrows*!

Whatever's at the top of the borrow stack is "live" and knows it's effectively unaliased. When you reborrow a pointer, the new pointer is pushed onto the stack, becoming *the* live pointer. When you use an older pointer it's brought back to life by popping everything on the borrow stack above it. At this point the pointer "knows" it was reborrowed and that the memory might have been modified, but that it once more has exclusive access -- no need to worry about little angry men.

So it's actually *always* ok to access a reborrowed pointer, because we can always pop everything above it. The real trouble is accessing a pointer that has already been popped off of the borrow stack -- then you've messed up.

Thankfully the design of the borrowchecker ensures that safe Rust programs follow these rules, as we saw in the above example, but the compiler generally views this problem "backwards" from the stacked borrows perspective. Instead of saying using `ref1` invalidates `ref2`, it insists that `ref2` *must* be valid for all its uses, and that `ref1` is the one messing things up by going out of turn.

De ahí «no se puede usar `*ref1` porque se tomó prestado de forma mutable». Es el mismo resultado (especialmente con tiempos de vida no léxicos), pero expresado de una manera probablemente más intuitiva.

¡Pero el comprobador de préstamos no puede ayudarnos cuando empezamos a usar punteros inseguros!





# Préstamos apilados inseguros

Queremos encontrar una forma de que los punteros inseguros participen en este sistema de préstamos apilados, aunque el compilador no pueda seguirlos correctamente. También queremos que el sistema sea bastante permisivo para que no resulte *demasiado* fácil estropearlo y provocar comportamiento indefinido.

Es un problema difícil y no sé cómo resolverlo, pero quienes trabajaron en los préstamos apilados idearon algo plausible y `miri` intenta implementarlo.

La idea general es que, cuando conviertes una referencia (o cualquier otro puntero seguro) en un puntero crudo, es *básicamente* como tomar un nuevo préstamo. El puntero crudo puede hacer entonces lo que quiera con esa memoria, y cuando el nuevo préstamo caduca ocurre lo mismo que con los nuevos préstamos normales.

Pero la pregunta es: ¿cuándo caduca ese nuevo préstamo? Probablemente un buen momento sea cuando vuelves a usar la referencia original. De lo contrario, las cosas no forman una pila bien anidada.

¡Pero espera, puedes convertir un puntero crudo *en* una referencia! ¡Y puedes copiar punteros crudos! ¿Qué ocurre si haces `&mut -> *mut -> &mut -> *mut` y después accedes al primer `*mut`? ¿Cómo demonios funcionan entonces los préstamos apilados?

¡Sinceramente, no lo sé! Por eso las cosas son complicadas. De hecho son *aún* más complicadas porque los préstamos apilados intentan ser más permisivos y permitir que más código inseguro funcione como esperarías. Por eso ejecuto el código con `miri`, para que me ayude a detectar errores.

De hecho, este desorden es la razón por la que `miri` tiene un modo aún más experimental y estricto: `-Zmiri-tag-raw-pointers`.

Para activarlo, tenemos que pasarlo mediante una variable de entorno `MIRIFLAGS`, así:

```text
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri test
```

O así en Windows, donde basta con establecer la variable globalmente:

```text
$env:MIRIFLAGS="-Zmiri-tag-raw-pointers"
cargo +nightly-2022-01-21 miri test
```

En general intentaremos cumplir este modo especialmente estricto para tener *aún más* confianza en nuestro trabajo. En cierto sentido también es más «sencillo», así que resulta mejor para experimentar y adquirir intuición sobre los préstamos apilados.




# Managing Stacked Borrows

Así que, al usar punteros crudos, intentaremos seguir una heurística sencilla y contundente que, con suerte, ofrezca un amplio margen de error:

**Cuando empieces a usar punteros crudos, intenta usar ÚNICAMENTE punteros crudos.**

Esto reduce al mínimo la posibilidad de perder accidentalmente el «permiso» del puntero crudo para acceder a la memoria.

> **NARRATOR:** this is oversimplified in two regards:
>
> 1. Safe pointers often assert more properties than just aliasing: the memory is allocated, it's aligned, it's large enough to fit the type of the pointee, the pointee is properly initialized, etc. So it's even more dangerous to wildly throw them around when things are in a dubious state.
>
> 2. Even if you stay in raw pointer land, you no puede just wildly alias any memory. Pointers are conceptually tied to specific "allocations" (which can be as granular as a local variable on the stack), and you're not supposed to take a pointer from one allocation, offset it, and then access memory in a different allocation. If this was allowed, there would *always* be the threat of tiny angry men *everywhere*. This is part of the reason "pointers are just integers" is a *problematic* viewpoint.

Aun así, queremos referencias seguras en nuestra *interfaz*, porque queremos construir una buena *abstracción segura* para que el usuario de nuestra lista no tenga que conocer estos detalles ni preocuparse por ellos.

Así que haremos lo siguiente:

1. Al principio de un método, usar las referencias de entrada para obtener nuestros punteros crudos.
2. Intentar usar únicamente punteros inseguros a partir de ese momento.
3. Volver a convertir nuestros punteros crudos en punteros seguros al final, si es necesario.

Pero los campos de nuestros tipos son privados, así que los mantendremos *completamente* como punteros crudos.

De hecho, parte del gran error que cometimos fue seguir usando `Box`. `Box` tiene una anotación especial que indica al compilador: «esto se parece mucho a `&mut`, porque posee ese puntero de forma exclusiva». ¡Y es verdad!

Pero el puntero crudo que conservábamos hacia el final de la lista apuntaba al interior de un `Box`, así que cada vez que accedemos normalmente al `Box` probablemente invalidamos el «nuevo préstamo» de ese puntero crudo. ☠

En la siguiente sección volveremos a nuestra verdadera forma y nos golpearemos la cabeza con un montón de ejemplos.


