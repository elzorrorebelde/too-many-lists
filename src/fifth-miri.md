# Miri

*ríe nerviosamente* Esto de `unsafe` es muy fácil; no sé por qué todo el mundo dice lo contrario. Nuestro programa funciona perfectamente.

> **NARRATOR:** 🙂

...right?

> **NARRATOR:** 🙂

Bueno, ahora estamos escribiendo código `unsafe`, así que el compilador no puede ayudarnos a detectar errores de la misma manera. Puede que las pruebas *simplemente* hayan funcionado, pero que en realidad estén haciendo algo no determinista. Algo parecido al comportamiento indefinido.

Pero ¿qué podemos hacer? Hemos forzado las ventanas y escapado del aula de `rustc`. Ahora nadie puede ayudarnos.

...Espera, ¿quién es esa persona de aspecto sospechoso en el callejón?

*«Oye, chico, ¿quieres interpretar código Rust?»*

Wh- no? Why,

*«Es una locura, puede comprobar que la ejecución dinámica real de tu programa cumple la semántica del modelo de memoria de Rust. Te vuela la cabeza...»*

What?

*«Comprueba si provocas comportamiento indefinido.»*

Supongo que podría probar un intérprete *una sola vez*.

*«Tienes `rustup` instalado, ¿verdad?»*

Claro que sí, ¡es *la* herramienta para mantener actualizada la cadena de herramientas de Rust!

```text
> rustup +nightly-2022-01-21 component add miri

info: syncing channel updates for 'nightly-2022-01-21-x86_64-pc-windows-msvc'
info: latest update on 2022-01-21, rust version 1.60.0-nightly (777bb86bc 2022-01-20)
info: downloading component 'cargo'
info: downloading component 'clippy'
info: downloading component 'rust-docs'
info: downloading component 'rust-std'
info: downloading component 'rustc'
info: downloading component 'rustfmt'
info: installing component 'cargo'
info: installing component 'clippy'
info: installing component 'rust-docs'
info: installing component 'rust-std'
info: installing component 'rustc'
info: installing component 'rustfmt'
info: downloading component 'miri'
info: installing component 'miri'
```

¿Qué acabas de instalar en mi ordenador?

*«Lo bueno»*

> **NARRADOR:** Hay algunas cosas extrañas con las versiones de la cadena de herramientas:
>
> La herramienta que estamos instalando, `miri`, trabaja estrechamente con los
> componentes internos de `rustc`, por lo que solo está disponible en cadenas
> de herramientas `nightly`.
>
> `+nightly-2022-01-21` indica a `rustup` que queremos instalar `miri` con la
> cadena `nightly` de Rust de esa fecha. Especifico una fecha concreta porque a
> veces `miri` se queda atrás y no se puede compilar con algunas versiones
> `nightly`. `rustup` descargará automáticamente la cadena que especifiquemos
> con `+` si todavía no está instalada.
>
> 2022-01-21 es una versión `nightly` que sé que admite `miri`, como puedes
> comprobar en [esta página de estado](https://rust-lang.github.io/rustup-components-history/).
> También puedes usar simplemente `+nightly` si te sientes afortunado.
>
> Cada vez que invoquemos `miri` mediante `cargo miri`, usaremos esta sintaxis
> `+` para indicar la cadena en la que instalamos `miri`. Si no quieres tener que
> especificarla cada vez, puedes usar [`rustup override set`](https://rust-lang.github.io/rustup/overrides.html).

```text
> cargo +nightly-2022-01-21 miri test

I will run `"cargo.exe" "install" "xargo"` to install
a recent enough xargo. Proceed? [Y/n]
```

¿¡PERO QUÉ DEMONIOS ES XARGO!?

*«Todo va bien, no te preocupes.»*

```text
> y

    Updating crates.io index
  Installing xargo v0.3.24
...
    Finished release [optimized] target(s) in 10.65s
  Installing C:\Users\ninte\.cargo\bin\xargo-check.exe
  Installing C:\Users\ninte\.cargo\bin\xargo.exe
   Installed package `xargo v0.3.24` (executables `xargo-check.exe`, `xargo.exe`)

I will run `"rustup" "component" "add" "rust-src"` to install 
the `rust-src` component for the selected toolchain. Proceed? [Y/n]
```

UH???

*«¿A quién no le gusta tener una copia del código fuente de Rust?»*

```text
> y

info: downloading component 'rust-src'
info: installing component 'rust-src'
```

*«Sí, ya está listo; aquí viene lo bueno.»*

```text
   Compiling lists v0.1.0 (C:\Users\ninte\dev\tmp\lists)
    Finished test [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests (lists-5cc11d9ee5c3e924.exe)

error: Undefined Behavior: trying to reborrow for Unique at alloc84055, 
       but parent tag <209678> does not have an appropriate item in 
       the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |
    = help: this indicates a potential bug in the program: 
      it performed an invalid operation, but the rules it 
      violated are still experimental
    = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information

    = note: inside `std::option::Option::<std::boxed::Box<fifth::Node<i32>>>::map::<i32, [closure@src\fifth.rs:31:30: 40:10]>` at \lib\rustlib\src\rust\library\core\src\option.rs:846:18

note: inside `fifth::List::<i32>::pop` at src\fifth.rs:31:9
   --> src\fifth.rs:31:9
    |
31  | /         self.head.take().map(|head| {
32  | |             let head = *head;
33  | |             self.head = head.next;
34  | |
...   |
39  | |             head.elem
40  | |         })
    | |__________^
note: inside `fifth::test::basics` at src\fifth.rs:74:20
   --> src\fifth.rs:74:20
    |
74  |         assert_eq!(list.pop(), Some(1));
    |                    ^^^^^^^^^^
note: inside closure at src\fifth.rs:62:5
   --> src\fifth.rs:62:5
    |
61  |       #[test]
    |       ------- in this procedural macro expansion
62  | /     fn basics() {
63  | |         let mut list = List::new();
64  | |
65  | |         // Check empty list behaves right
...   |
96  | |         assert_eq!(list.pop(), None);
97  | |     }
    | |_____^
 ...
error: aborting due to previous error
```

Vaya. Ese es un error monumental.

*«Sí, mira eso. Da gusto verlo.»*

¿Gracias?

*«Toma también esta botella de estradiol; la necesitarás más adelante.»*

¿Por qué?

*«Estás a punto de empezar a pensar en modelos de memoria, créeme.»*

> **NARRADOR:** La misteriosa persona se transformó entonces en un zorro y salió corriendo por un agujero de la pared. El autor se quedó mirando al vacío durante varios minutos mientras intentaba asimilar todo lo que acababa de ocurrir.


-------

El misterioso zorro del callejón tenía razón en algo más que en lo relacionado con mi género: `miri` es realmente la leche.

De acuerdo, ¿qué *es* [miri](https://github.com/rust-lang/miri)?

> Un intérprete experimental de la representación intermedia de nivel medio de Rust (MIR). Puede ejecutar binarios y conjuntos de pruebas de proyectos de Cargo y detectar ciertas clases de comportamiento indefinido, por ejemplo:
>
> * Accesos a memoria fuera de los límites y uso después de liberar memoria.
> * Uso incorrecto de datos no inicializados.
> * Incumplimiento de precondiciones intrínsecas (alcanzar `unreachable_unchecked`, llamar a `copy_nonoverlapping` con rangos solapados, ...).
> * Accesos y referencias a memoria con una alineación insuficiente.
> * Incumplimiento de algunas invariantes básicas de los tipos (por ejemplo, un `bool` que no sea 0 o 1, o un discriminante de `enum` no válido).
> * Experimental: incumplimientos de las reglas de préstamos apilados que gobiernan el aliasing de los tipos de referencia.
> * Experimental: carreras de datos (pero no efectos de memoria débil).
>
> Además, `Miri` también te informará de las fugas de memoria: si queda memoria asignada al final de la ejecución y esa memoria no es accesible desde un `static` global, `Miri` emitirá un error.
>
> ...
>
> Sin embargo, ten en cuenta que `Miri` no detecta todos los casos de comportamiento indefinido de tu programa y no puede ejecutar todos los programas.

En resumen: interpreta tu programa y detecta si infringe las reglas *en tiempo de ejecución* y provoca comportamiento indefinido. Esto es necesario porque el comportamiento indefinido suele producirse en tiempo de ejecución. Si el problema pudiera detectarse al compilar, ¡el compilador simplemente lo convertiría en un error!

Si conoces herramientas como `ubsan` y `tsan`, esto es algo parecido, pero todo junto y de forma más extrema.

-------

Ahora `miri` cuelga fuera de la ventana del aula con un cuchillo. Un cuchillo didáctico.

Si queremos que `miri` compruebe nuestro trabajo, podemos pedirle que interprete nuestro conjunto de pruebas con

```text
> cargo +nightly-2022-01-21 miri test
```

Now let's take a closer look at what they carved into our desk:

```text
error: Undefined Behavior: trying to reborrow for Unique at alloc84055, but parent tag <209678> does not have an appropriate item in the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |

    = help: this indicates a potential bug in the program: it 
      performed an invalid operation, but the rules it 
      violated are still experimental
    
    = help: see 
      https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information
```

Veo que hemos cometido un error, pero es un mensaje bastante confuso. ¿Qué es la «pila de préstamos»?

Intentaremos averiguarlo en la siguiente sección.