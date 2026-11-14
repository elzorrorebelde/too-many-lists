# Una Doble Cola Insegura de Producción Doblemente Enlazada

Finalmente lo logramos. Mi gran némesis: **[std::collections::LinkedList][linked-list], la Doble Cola Doblemente Enlazada**.

La que intenté destruir y fallé.

Nuestra historia comienza cuando 2014 llegaba a su fin y nos acercábamos
rápidamente al lanzamiento de Rust 1.0, la primera versión estable de Rust.
Me había encontrado en el rol de cuidar de `std::collections`, o como lo
llamábamos cariñosamente en esos tiempos, libcollections.

libcollections había pasado años como vertedero para las Ideas Lindas y Cosas
Vagamente Útiles de todos. Esto estaba bien cuando Rust era un lenguaje
experimental incipiente, pero si mis hijos iban a escapar del nido y ser
estabilizados, tendrían que demostrar su valía.

Hasta entonces los había alentado y nutrido a todos, pero ahora era hora de
que enfrentaran el juicio por sus fallos.

Hundí mis garras en la roca base y tallé lápidas para mis hijos más tontos.
Un monumento macabro que coloqué en la plaza del pueblo para que todos lo
vieran:

**[Kill TreeMap, TreeSet, TrieMap, TrieSet, LruCache and EnumSet](https://github.com/rust-lang/rust/pull/19955)**

Sus destinos estaban sellados, porque mi palabra era absoluta. Las otras
colecciones estaban horrorizadas por mi brutalidad, pero aún no estaban a
salvo de la ira de su madre. Pronto regresé con dos lápidas más:

**[Deprecate BitSet and BitVec](https://github.com/rust-lang/rust/pull/26034)**

Los gemelos Bit eran más astutos que sus camaradas caídos, pero les faltaba
la fuerza para escaparme. La mayoría pensó que mi trabajo había terminado,
pero pronto tomé una más:

**[Deprecate VecMap](https://github.com/rust-lang/rust/pull/26734)**

VecMap había intentado sobrevivir mediante el sigilo &mdash; ¡era tan pequeño
e inofensivo! Pero eso no era suficiente para el libcollections que veía en mi
visión del futuro.

Inspeccioné la tierra y vi lo que quedaba:

* Vec y VecDeque — robustos y simples, el corazón de la computación.
* HashMap y HashSet — poderosos y sabios, el cerebro de la computación.
* BTreeMap y BTreeSet — torpes pero necesarios, el hígado de la computación.
* BinaryHeap — astuto y diestro, el tobillo de la computación.

Asentí con satisfacción. Simple y efectivo. Mi trabajo estaba hech&mdash;

No, ¡[DList](https://github.com/rust-lang/rust/blob/0a84308ebaaafb8fd89b2fd7c235198e3ec21384/src/libcollections/dlist.rs), no puede ser! ¡Pensé que habías muerto en ese trágico incidente de recolección de basura! ¡Ese que definitivamente fue un accidente y no intencional en absoluto!

Habían fingido su muerte y adoptado un nuevo nombre, pero seguían siendo ellos: LinkedList, el maquinador sombrío y poco confiable de la computación.

Difundí la palabra de sus fechorías a todos los que me escucharan, pero los
corazones no se conmovieron. LinkedList era un diablo de lengua plateada que
había convencido a todos a mi alrededor de que era una especie de estructura
de datos fundamental y natural de la computación. ¡Incluso había convencido a
C++ de que era [*la* lista](https://en.cppreference.com/w/cpp/container/list)!

"¿Cómo podrías tener una biblioteca estándar sin un *LinkedList*?"

¡Fácilmente! ¡Triamente!

"¡Es código inseguro no trivial, así que tiene sentido tenerlo en la biblioteca estándar!"

¡También lo son los controladores de GPU y los códecs de video, y libcollections es minimalista!

Pero lamentablemente, LinkedList había reunido demasiados aliados y se había
vuelto demasiado fuerte mientras yo estaba distraído con sus parientes.

Huí a mi laboratorio e intenté idear algún tipo de [clon maligno](https://github.com/contain-rs/linked-list) o [replicante cyborg mejorado](https://github.com/contain-rs/blist) que pudiera rivalizar y destruirlo, pero mi financiación se agotó porque mi investigación era "demasiado asesinamente malvada" o alguna tontería similar.

LinkedList había ganado. Fui derrotado y forzado al exilio.

Pero estás aquí ahora. Has llegado hasta aquí. Seguramente ahora puedes
entender las profundidades de la depravación de LinkedList. Ven, te mostraré
todo lo que necesitas saber para ayudarme a destruirlo de una vez por todas
&mdash; todo lo que necesitas saber para implementar una Doble Cola Doblemente
Enlazada insegura de calidad de producción.

¿Qué tan calidad de producción? Bueno, vamos a reescribir completamente mi
antiguo crate de lista enlazada de Rust 1.0, el que es objetivamente mejor que
el de std. ¡El que tiene Cursores en Rust estable, desde 2015! ¡Algo que la
stdlib de 2022 todavía no tiene!





[linked-list]: https://github.com/rust-lang/rust/blob/master/library/alloc/src/collections/linked_list.rs