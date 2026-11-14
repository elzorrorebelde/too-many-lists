# Rust Inseguro (Unsafe Rust)

Este es un tema serio, grande, complicado y peligroso.
Es tan serio que escribí [otro libro completo][nom] sobre ello.

El resumen es que *todo* lenguaje es en realidad inseguro tan pronto como
permites llamar a otros lenguajes, porque puedes hacer que C haga cosas
arbitrariamente malas. Sí: Java, Python, Ruby, Haskell... todos son
salvajemente inseguros frente a Interfaces de Funciones Extranjeras (FFI).

Rust abraza esta verdad dividiéndose en dos lenguajes: Rust Seguro y Rust
Inseguro. Hasta ahora solo hemos trabajado con Rust Seguro. Es completamente
100% seguro... excepto que puede hacer FFI a Rust Inseguro.

Rust Inseguro es un *superconjunto* de Rust Seguro. Es completamente igual a
Rust Seguro en toda su semántica y reglas, solo que se te permiten hacer
algunas cosas *extra* que son salvajemente inseguras y pueden causar el temido
Comportamiento Indefinido (Undefined Behaviour) que acecha a C.

De nuevo, este es un tema realmente enorme que tiene muchos casos
interesantes. Realmente *no* quiero profundizar mucho en ello (bueno, sí
quiero. Lo hice. [Lee ese libro][nom]). Está bien, porque con las listas
enlazadas podemos ignorar casi todo eso.

> **NARRADOR:** Esto era mentira, pero parecía cierto en 2015.

La principal herramienta Insegura que usaremos son los *punteros crudos (raw
pointers)*. Los punteros crudos son básicamente los punteros de C. No tienen
reglas de aliasing inherentes. No tienen tiempos de vida. Pueden ser nulos.
Pueden estar desalineados. Pueden colgar. Pueden apuntar a memoria no
inicializada. Pueden convertirse a y desde enteros. Pueden convertirse para
apuntar a un tipo diferente. ¿Mutabilidad? Conviértelo. Casi todo vale, y eso
significa que casi cualquier cosa puede salir mal.

> **NARRADOR:** Sin reglas de aliasing inherentes, eh? Ah, la inocencia de la
> juventud.

Esto es material malo y honestamente vivirías una vida más feliz sin tener que
tocar esto nunca. Desafortunadamente, queremos escribir listas enlazadas, y las
listas enlazadas son horribles. Eso significa que vamos a tener que usar
punteros inseguros.

Hay dos tipos de puntero crudo: `*const T` y `*mut T`. Se supone que son
`const T*` y `T*` de C, pero realmente no nos importa lo que C piense que
significan. Solo puedes desreferenciar un `*const T` a un `&T`, pero al igual
que la mutabilidad de una variable, esto es solo una advertencia contra uso
incorrecto. Como mucho significa que tienes que convertir el `*const` a un
`*mut` primero. Aunque si realmente no tienes permiso para mutar el referente
del puntero, la vas a pasar mal.

Como sea, entenderemos mejor esto mientras escribamos algo de código. Por
ahora, `*mut T == &unchecked mut T`!

[nom]: https://doc.rust-lang.org/nightly/nomicon/