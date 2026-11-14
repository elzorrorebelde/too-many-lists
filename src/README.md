# Aprendiendo Rust Con Demasiadas Listas Enlazadas

> ¿Tienes algún problema o quieres ver todo el código final de una vez?
> [¡Todo está en Github!][github]

> **NOTA**: La edición actual de este libro está escrita para Rust 2018,
> que se lanzó por primera vez con rustc 1.31 (8 de diciembre de 2018). Si tu
> cadena de herramientas de Rust es suficientemente nueva, el archivo
> `Cargo.toml` que `cargo new` crea debería contener la línea
> `edition = "2018"` (o si estás leyendo esto en un futuro lejano, ¡quizás
> un número aún mayor!). Usar una cadena de herramientas más antigua es posible,
> pero desbloquea un **modo difícil** secreto, donde obtendrás errores extra del
> compilador que no se mencionan en el texto de este libro. ¡Vaya, suena divertido!

Con bastante frecuencia me preguntan cómo implementar una lista enlazada en Rust.
La respuesta honestamente depende de cuáles sean tus requisitos, y obviamente no
es muy fácil responder la pregunta en el momento. Por eso he decidido escribir
este libro para responder la pregunta de forma exhaustiva de una vez por todas.

En esta serie te enseñaré programación básica y avanzada de Rust
íntegramente haciéndote implementar 6 listas enlazadas. Al hacerlo, deberías
aprender:

* Los siguientes tipos de punteros: `&`, `&mut`, `Box`, `Rc`, `Arc`, `*const`, `*mut`, `NonNull`(?)
* Propiedad (ownership), préstamo (borrowing), mutabilidad heredada, mutabilidad interior, `Copy`
* Todas las palabras clave: struct, enum, fn, pub, impl, use, ...
* Pattern matching, genéricos, destructores
* Pruebas, instalación de nuevas cadenas de herramientas, uso de `miri`
* Rust inseguro: punteros crudos, aliasing, stacked borrows, `UnsafeCell`, varianza

Sí, las listas enlazadas son tan verdaderamente horribles que terminas lidiando
con todos estos conceptos al hacerlas reales.

Todo está en la barra lateral (puede estar colapsada en móvil), pero para
referencia rápida, aquí está lo que vamos a hacer:

1. [Una Pila Mala Enlazada Individualmente](first.md)
2. [Una Pila Aceptable Enlazada Individualmente](second.md)
3. [Una Pila Persistente Enlazada Individualmente](third.md)
4. [Una Doble Cola Mala Pero Segura Doblemente Enlazada](fourth.md)
5. [Una Cola Aceptable Insegura Enlazada Individualmente](fifth.md)
6. [Una Doble Cola Insegura de Producción Doblemente Enlazada](sixth.md)
7. [Bonus: Un Montón de Listas Tontas](infinity.md)

Solo para que estemos todos en la misma página, escribiré todos los comandos
que introduzco en mi terminal. También usaré el gestor de paquetes estándar de Rust,
Cargo, para desarrollar el proyecto. Cargo no es necesario para escribir un
programa en Rust, pero es *mucho* mejor que usar rustc directamente. Si solo
quieres jugar un poco también puedes ejecutar algunos programas simples en el
navegador mediante [play.rust-lang.org][play].

En secciones posteriores, usaremos "rustup" para instalar herramientas
adicionales de Rust. Recomiendo encarecidamente
[instalar todas tus cadenas de herramientas de Rust usando rustup](https://www.rust-lang.org/tools/install).

Empecemos y creemos nuestro proyecto:

```text
> cargo new --lib lists
> cd lists
```

Pondremos cada lista en un archivo separado para no perder nada de nuestro trabajo.

Cabe señalar que la experiencia *auténtica* de aprender Rust implica
escribir código, que el compilador te grite, e intentar averiguar
qué demonios significa eso. Me aseguraré cuidadosamente de que esto ocurra
con la mayor frecuencia posible. Aprender a leer y entender los errores
generalmente excelentes del compilador de Rust y su documentación es
*increíblemente* importante para ser un programador productivo de Rust.

Aunque en realidad eso es mentira. Mientras escribía esto encontré *muchos*
más errores del compilador de los que muestro. En particular, en los capítulos
posteriores no mostraré muchos de los errores aleatorios de "escribí mal
(copié y pegué)" que esperas encontrar en cualquier lenguaje. Esto es un *tour
guiado* de tener al compilador gritándonos.

Iremos bastante lento, y honestamente no seré muy serio prácticamente todo
el tiempo. ¡Creo que programar debería ser divertido, caramba!
Si eres del tipo de persona que quiere contenido máximamente denso en
información, serio y formal, este libro no es para ti. Nada de lo que
haga será jamás para ti. Estás equivocado.




# Un Anuncio de Servicio Público Obligatorio

Solo para que estemos total y absolutamente claros: odio las listas enlazadas. Con
pasión. Las listas enlazadas son estructuras de datos terribles. Ahora bien, por
supuesto hay varios casos de uso excelentes para una lista enlazada:

* Quieres hacer *mucho*分割 o fusión de listas grandes. *Mucho*.
* Estás haciendo algo genial de concurrencia sin bloqueos (lock-free).
* Estás escribiendo un núcleo/sistema embebido y quieres usar una lista intrusiva.
* Estás usando un lenguaje puramente funcional y la semántica limitada y la
  ausencia de mutación hacen que las listas enlazadas sean más fáciles de usar.
* ... ¡y más!

Pero todos estos casos son *super raros* para cualquiera que escriba un programa
en Rust. El 99% del tiempo deberías usar un `Vec` (array/pila), y el 99% del
otro 1% del tiempo deberías usar un `VecDeque` (array/deque). Estas son
estructuras de datos abiertamente superiores para la mayoría de las cargas de
trabajo debido a una asignación de memoria menos frecuente, menor sobrecarga de
memoria, acceso aleatorio real y localidad de caché.

Las listas enlazadas son una estructura de datos tan *especializada* y *vaga*
como un trie. Pocos se sorprenderían si afirmara que un trie es una estructura
especializada que un programador promedio podría felizmente nunca aprender en
toda una carrera productiva — y sin embargo las listas enlazadas tienen un
extraño estatus de celebridad. Enseñamos a cada estudiante a escribir una lista
enlazada. Es la única colección especializada
[que no pude eliminar de std::collections][rust-std-list]. Es
[*la* lista en C++][cpp-std-list]!

Deberíamos todos como comunidad decir *no* a las listas enlazadas como una
estructura de datos "estándar". Es una estructura de datos fine con varios
casos de uso excelentes, pero esos casos de uso son *excepcionales*, no comunes.

Varias personas aparentemente leen el primer párrafo de este PSA y luego dejan
de leer. Como, literalmente intentan refutar mi argumento enumerando una de las
cosas en mi lista de *casos de uso excelentes*. ¡Justo lo que viene después del
primer párrafo!

Solo para poder enlazar directamente a un argumento detallado, aquí hay varios
intentos de contraargumentos que he visto, y mi respuesta a ellos. Siéntete libre
de saltar al [primer capítulo](first.md) si solo quieres aprender algo de Rust.




## El rendimiento no siempre importa

¡Sí! Quizás tu aplicación está limitada por E/S o el código en cuestión está en
algún caso poco relevante que simplemente no importa. Pero esto ni siquiera es un
argumento para usar una lista enlazada. Es un argumento para usar *lo que sea*.
¿Por qué conformarte con una lista enlazada? ¡Usa un mapa hash enlazado!

Si el rendimiento no importa, entonces *seguramente* está bien aplicar la opción
natural por defecto de un array.





## Tienen split-append-insert-remove O(1) si tienes un puntero allí

¡Sip! Aunque como [señala Bjarne Stroustrup][bjarne] *esto en realidad no
importa* si el tiempo que toma obtener ese puntero supera por completo el
tiempo que tomaría simplemente copiar todos los elementos en un array (que
es realmente bastante rápido).

A menos que tengas una carga de trabajo que esté fuertemente dominada por
costos de dividir y fusionar, la penalización que *todas las demás*
operaciones sufren debido a efectos de caché y complejidad del código
eliminará cualquier ganancia teórica.

*Pero sí, si estás perfilando tu aplicación y esta pasa mucho tiempo en
dividir y fusionar, puedes tener ganancias con una lista enlazada*.





## No puedo permitirme la amortización

Ya has entrado en un espacio bastante especializado — la mayoría puede
permitirse la amortización. Aún así, los arrays se amortizan *en el peor caso*.
Solo porque uses un array, no significa que tengas costos amortizados. Si puedes
predecir cuántos elementos vas a almacenar (o incluso tener un límite superior),
puedes pre-reservar todo el espacio que necesites. En mi experiencia es *muy*
común poder predecir cuántos elementos necesitarás. En Rust en particular, todos
los iteradores proporcionan un `size_hint` exactamente para este caso.

Entonces `push` y `pop` serán operaciones verdaderamente O(1). Y van a ser
*considerablemente* más rápidas que `push` y `pop` en una lista enlazada.
Haces un desplazamiento de puntero, escribes los bytes, e incrementas un
entero. Sin necesidad de ir a ningún tipo de asignador.

¿Qué tal para baja latencia?

*Pero sí, si no puedes predecir tu carga, ¡hay ahorros de latencia en el peor
caso que puedes obtener!*





## Las listas enlazadas desperdician menos espacio

Bueno, esto es complicado. Una estrategia de "redimensionamiento" estándar de
un array es crecer o encoger para que como máximo la mitad del array esté vacía.
Esto es de hecho una gran cantidad de espacio desperdiciado. Especialmente en Rust,
no encogemos colecciones automáticamente (es un desperdicio si solo vas a
llenarlo de nuevo), por lo que el desperdicio puede acercarse al infinito.

Pero esto es un escenario de peor caso. En el mejor caso, una pila con array
solo tiene tres punteros de sobrecarga para todo el array. Básicamente sin
sobrecarga.

Las listas enlazadas, por otro lado, desperdician espacio incondicionalmente
por elemento. Una lista enlazada individualmente desperdicia un puntero mientras
que una doblemente enlazada desperdicia dos. A diferencia de un array, el
desperdicio relativo es proporcional al tamaño del elemento. Si tienes elementos
*enormes* esto se acerca al 0% de desperdicio. Si tienes elementos diminutos
(digamos, bytes), entonces esto puede ser hasta 16 veces la sobrecarga de
memoria (8 veces en 32 bits).

En realidad, es más como 23 veces (11 veces en 32 bits) porque se añadirá
relleno (padding) al byte para alinear el tamaño total del nodo a un puntero.

Esto también asume el mejor caso para tu asignador: que asignar y
desasignar nodos se hace de forma densa y no estás perdiendo memoria por
fragmentación.

*Pero sí, si tienes elementos enormes, no puedes predecir tu carga, y tienes
un asignador decente, ¡hay ahorros de memoria que puedes obtener!*





## Uso listas enlazadas todo el tiempo en &lt;lenguaje funcional&gt;

¡Genial! Las listas enlazadas son super elegantes de usar en lenguajes
funcionales porque puedes manipularlas sin ninguna mutación, puedes describirlas
recursivamente, y también trabajar con listas infinitas gracias a la magia de la
pereza (laziness).

Específicamente, las listas enlazadas son agradables porque representan una
iteración sin necesidad de ningún estado mutable. El siguiente paso es
simplemente visitar la siguiente sublista.

Rust hace este tipo de cosas principalmente con [iteradores][iterators]. Pueden
ser infinitos y puedes mapearlos, filtrarlos, invertirlos y concatenarlos como
una lista funcional, ¡y todo se hará igual de perezosamente!

Rust también te permite hablar fácilmente de sub-arrays con *[rebanadas
(slices)][slices]*. Tu división habitual cabeza/cola en un lenguaje funcional
es [solo `slice.split_at_mut(1)`][split]. Durante mucho tiempo, Rust tuvo un
sistema experimental para pattern matching en slices que era super genial,
pero la característica se simplificó cuando se estabilizó. Aún así,
[los patrones básicos de slices][slice-pats] son interesantes. Y por supuesto,
¡los slices se pueden convertir en iteradores!

*Pero sí, si estás limitado a semántica inmutable, las listas enlazadas pueden ser
muy agradables*.

Nótese que no estoy diciendo que la programación funcional sea necesariamente
débil o mala. Sin embargo, *es* fundamentalmente limitada semánticamente: en
gran medida solo se te permite hablar de cómo *son* las cosas, y no de cómo
deberían *hacerse*. Esto es en realidad una *característica*, porque permite
al compilador hacer toneladas de [transformaciones exóticas][ghc] y
potencialmente encontrar la *mejor* manera de hacer las cosas sin que tengas
que preocuparte. Sin embargo, esto tiene el costo de *poder* preocuparte por
ello. Generalmente hay escapes, pero en algún límite solo estás escribiendo
código procedural otra vez.

Incluso en lenguajes funcionales, deberías esforzarte por usar la estructura
de datos apropiada para el trabajo cuando realmente necesitas una estructura
de datos. Sí, las listas enlazadas individualmente son tu herramienta principal
para el flujo de control, pero son una forma realmente pobre de almacenar un
montón de datos y consultarlos.


## ¡Las listas enlazadas son geniales para construir estructuras de datos concurrentes!

¡Sí! Aunque escribir una estructura de datos concurrente es realmente un animal
completamente diferente, y no es algo que deba tomarse a la ligera. Ciertamente
no es algo que mucha gente siquiera *considerará* hacer. Una vez que se ha
escrito, tampoco estás realmente eligiendo usar una lista enlazada. Estás
eligiendo usar una cola MPSC o lo que sea. ¡La estrategia de implementación
está bastante lejos en este caso!

*Pero sí, las listas enlazadas son las heroínas de facto del oscuro mundo de la
concurrencia sin bloqueos.*




## Mumble mumble kernel embebido algo algo intrusivo.

Es especializado. Estás hablando de una situación donde ni siquiera estás
usando el *entorno de ejecución* de tu lenguaje. ¿No es esa una bandera roja
de que estás haciendo algo extraño?

También es salvajemente inseguro.

*Pero claro. Construye tus increíbles listas de cero asignaciones en la pila.*





## Los iteradores no se invalidan por inserciones/eliminaciones no relacionadas

Esa es una danza delicada la que estás realizando. Especialmente si no tienes
un recolector de basura. Podría argumentar que tus patrones de flujo de control
y propiedad están probablemente un poco demasiado enredados, dependiendo de los
detalles.

*Pero sí, puedes hacer cosas realmente geniales y locas con cursores.*





## ¡Son simples y geniales para enseñar!

Bueno, sí. Estás leyendo un libro dedicado a esa premisa.
Bueno, las listas enlazadas individualmente son bastante simples. Las listas
doblemente enlazadas pueden ponerse bastante complicadas, como veremos.




# Toma un Respiro

Ok. Eso ya está fuera del camino. Escribamos un montón de listas enlazadas.

[¡Al primer capítulo!](first.md)


[rust-std-list]: https://doc.rust-lang.org/std/collections/struct.LinkedList.html
[cpp-std-list]: http://en.cppreference.com/w/cpp/container/list
[github]: https://github.com/elzorrorebelde/too-many-lists
[bjarne]: https://web.archive.org/web/20240228123439/https://www.youtube.com/watch?v=YQs6IC-vgmo
[slices]: https://doc.rust-lang.org/std/primitive.slice.html
[split]: https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut
[slice-pats]: https://doc.rust-lang.org/edition-guide/rust-2018/slice-patterns.html
[iterators]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[ghc]: https://wiki.haskell.org/GHC_optimisations#Fusion
[play]: https://play.rust-lang.org/