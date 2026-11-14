# Drop

Podemos crear una pila, hacer push, hacer pop, ¡e incluso hemos probado que
todo funciona correctamente!

¿Necesitamos preocuparnos por limpiar nuestra lista? Técnicamente, ¡no, en
absoluto! Como C++, Rust usa destructores para limpiar automáticamente los
recursos cuando ya no se necesitan. Un tipo tiene un destructor si implementa
un *trait* llamado Drop. Los traits son el término elegante de Rust para las
interfaces. El trait Drop tiene la siguiente interfaz:

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

Básicamente, "cuando salgas del ámbito (scope), te daré un segundo para que
ordene tus asuntos".

En realidad no necesitas implementar Drop si contienes tipos que implementan
Drop, y todo lo que querrías hacer es llamar a *sus* destructores. En el caso
de List, todo lo que querría hacer es soltar su cabeza, que a su vez
*quizás* intentaría soltar un `Box<Node>`. Todo eso se maneja por nosotros
automáticamente... con un problema.

El manejo automático va a ser malo.

Consideremos una lista simple:

```text
list -> A -> B -> C
```

Cuando se suelta `list`, intentará soltar A, que intentará soltar B,
que intentará soltar C. Algunos podrían estar poniéndose nerviosos con razón.
¡Esto es código recursivo, y el código recursivo puede reventar la pila!

Algunos podrían estar pensando "esto es claramente recursivo por la cola
(tail recursive), y cualquier lenguaje decente se aseguraría de que tal código
no reventara la pila". ¡Esto es, de hecho, incorrecto! Para ver por qué,
intentemos escribir lo que el compilador tiene que hacer, implementando
manualmente Drop para nuestra List como lo haría el compilador:


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTA: en realidad no puedes llamar explícitamente a `drop` en Rust;
        // ¡estamos pretendiendo ser el compilador!
        self.head.drop(); // tail recursive - bien!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // ¡Hecho!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - bien!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, no es tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

*No podemos* soltar el contenido del Box *después* de desasignarlo, así que no
hay forma de soltar de manera recursiva por la cola. En su lugar, vamos a
tener que escribir manualmente un drop iterativo para `List` que extraiga los
nodos de sus cajas (boxes).


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "haz esto hasta que este patrón no coincida"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node sale del ámbito y se suelta aquí;
            // pero el campo `next` de su Node se ha puesto a Link::Empty
            // así que no ocurre recursión sin límites.
        }
    }
}
```

Con nuestra implementación personalizada de drop, ahora ambas pruebas funcionan.

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test first::test::long_list ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

¡Genial!

----------------------

<span style="float:left">![Bonus](img/profbee.gif)</span>

## Sección Extra para Optimización Prematura!

Nuestra implementación de drop es en realidad *muy* similar a
`while let Some(_) = self.pop() { }`, que es ciertamente más simple. ¿Cómo
es diferente, y qué problemas de rendimiento podrían resultar de ella una vez
que comencemos a generalizar nuestra lista para almacenar cosas que no sean
enteros?

<details>
  <summary>Haz clic para expandir la respuesta</summary>

Pop devuelve `Option<i32>`, mientras que nuestra implementación solo manipula Links (`Box<Node>`). Así que nuestra implementación solo mueve punteros a nodos, mientras que la basada en pop moverá los valores que almacenamos en los nodos. Esto podría ser muy costoso si generalizamos nuestra lista y alguien la usa para almacenar instancias de VeryBigThingWithADropImpl (VBTWADI). Box puede ejecutar la implementación de drop de sus contenidos in-situ, por lo que no sufre de este problema. Como VBTWADI es *exactamente* el tipo de cosa que realmente hace que usar una lista enlazada sea deseable sobre un array, comportarse mal en este caso sería una decepción.

Si deseas tener lo mejor de ambas implementaciones, podrías añadir un nuevo método,
`fn pop_node(&mut self) -> Link`, del cual tanto `pop` como `drop` pueden
derivarse limpiamente.

</details>