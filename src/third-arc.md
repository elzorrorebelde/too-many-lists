# Arc

Una razón para usar una lista enlazada inmutable es compartir datos entre
hilos. Después de todo, el estado mutable compartido es la raíz de todos los
males, y una forma de resolverlo es matar la parte *mutable* para siempre.

Excepto que nuestra lista no es para nada segura para hilos (thread-safe).
Para ser segura para hilos, necesitamos manipular los contadores de referencias
*atómicamente*. De lo contrario, dos hilos podrían intentar incrementar el
contador de referencias, *y solo uno ocurriría*. ¡Entonces la lista podría
liberarse demasiado pronto!

Para obtener seguridad para hilos, tenemos que usar *Arc*. Arc es
completamente idéntico a Rc excepto por el hecho de que los contadores de
referencias se modifican atómicamente. Esto tiene un poco de sobrecarga si no
lo necesitas, así que Rust expone ambos. Todo lo que necesitamos hacer para
hacer nuestra lista segura para hilos es reemplazar cada referencia a Rc con
`std::sync::Arc`. Eso es todo. Somos seguros para hilos. ¡Hecho!

Pero esto plantea una pregunta interesante: ¿cómo *sabemos* si un tipo es
seguro para hilos o no? ¿Podemos estropearlo accidentalmente?

¡No! ¡No puedes estropear la seguridad para hilos en Rust!

La razón por la que esto es así es porque Rust modela la seguridad para hilos
de primera clase con dos traits: `Send` y `Sync`.

Un tipo es *Send* si es seguro *moverlo* a otro hilo. Un tipo es *Sync* si es
seguro *compartirlo* entre múltiples hilos. Es decir, si `T` es Sync, `&T` es
Send. Seguro en este caso significa que es imposible causar *data races* (que
no debe confundirse con el problema más general de *race conditions*).

Estos son traits marcadores (marker traits), que es una forma elegante de
decir que son traits que no proporcionan absolutamente ninguna interfaz. O
*eres* Send, o no lo eres. Es solo una propiedad que otras *APIs* pueden
exigir. ¡Si no eres apropiadamente Send, entonces es estáticamente imposible
ser enviado a un hilo diferente! ¡Genial!

Send y Sync también son traits que se derivan automáticamente basándose en si
estás totalmente compuesto de tipos Send y Sync. Es similar a cómo solo
puedes implementar Copy si estás hecho solo de tipos Copy, pero entonces
simplemente lo implementamos automáticamente si lo eres.

Casi todos los tipos son Send y Sync. La mayoría de los tipos son Send porque
poseen totalmente sus datos. La mayoría de los tipos son Sync porque la única
forma de compartir datos entre hilos es ponerlos detrás de una referencia
compartida, ¡lo que los hace inmutables!

Sin embargo, hay tipos especiales que violan estas propiedades: aquellos que
tienen *mutabilidad interior (interior mutability)*. Hasta ahora solo hemos
interactuado realmente con *mutabilidad heredada (inherited mutability)*
(también conocida como mutabilidad externa): la mutabilidad de un valor se
hereda de la mutabilidad de su contenedor. Es decir, no puedes simplemente
mutar aleatoriamente algún campo de un valor no mutable porque te apetezca.

Los tipos de mutabilidad interior violan esto: te permiten mutar a través de
una referencia compartida. Hay dos clases principales de mutabilidad interior:
cells (celdas), que solo funcionan en un contexto de un solo hilo; y locks
(cerrojos), que funcionan en un contexto de múltiples hilos. Por razones
obvias, las cells son más baratas cuando puedes usarlas. También están los
atómicos, que son primitivos que actúan como un lock.

Entonces, ¿qué tiene que ver todo esto con Rc y Arc? Bueno, ambos usan
mutabilidad interior para su *contador de referencias*. Peor aún, este
contador de referencias se comparte entre cada instancia. Rc simplemente usa
una cell, lo que significa que no es seguro para hilos. Arc usa un atómico,
lo que significa que *sí* es seguro para hilos. Por supuesto, no puedes
magicamente hacer que un tipo sea seguro para hilos poniéndolo en Arc. Arc
solo puede derivar seguridad para hilos como cualquier otro tipo.

Realmente, realmente, realmente no quiero entrar en los detalles más finos de
los modelos de memoria atómicos o implementaciones de Send no derivadas. No
hace falta decir que, a medida que te adentras en la historia de seguridad
para hilos de Rust, las cosas se complican más. Como consumidor de alto nivel,
todo *simplemente funciona* y realmente no necesitas pensar en ello.