# Propiedad (Ownership) 101

Ahora que podemos construir una lista, estaría bien poder *hacer* algo
con ella. Hacemos eso con métodos "normales" (no estáticos). Los métodos son un
caso especial de función en Rust por el argumento `self`, que no tiene
un tipo declarado:

```rust ,ignore
fn foo(self, arg2: Type2) -> ReturnType {
    // body
}
```

Hay 3 formas principales que self puede tomar: `self`, `&mut self` y `&self`.
Estas 3 formas representan las tres formas principales de propiedad (ownership)
en Rust:

* `self` — Valor
* `&mut self` — Referencia mutable
* `&self` — Referencia compartida

Un valor representa *propiedad verdadera*. Puedes hacer lo que quieras con un valor:
moverlo, destruirlo, mutarlo, o prestarlo mediante una referencia. Cuando pasas
algo por valor, se *mueve* a la nueva ubicación. La nueva ubicación ahora
posee el valor, y la ubicación anterior ya no puede acceder a él. Por esta razón,
la mayoría de los métodos no quieren `self` — sería bastante malo si intentar
trabajar con una lista hiciera que desapareciera.

Una referencia mutable representa *acceso exclusivo* temporal a un valor que no
posees. Puedes hacer absolutamente cualquier cosa que quieras a un valor del que
tienes una referencia mutable, siempre que lo dejes en un estado válido cuando
termines (¡sería grosero con el dueño si no!). Esto significa que puedes incluso
sobrescribir completamente el valor. Un caso especial realmente útil de esto es
*intercambiar* un valor por otro, que usaremos mucho. La única cosa que no puedes
hacer con un `&mut` es mover el valor sin reemplazo. `&mut self` es genial para
métodos que quieren mutar `self`.

Una referencia compartida representa *acceso compartido* temporal a un valor que no
posees. Como tienes acceso compartido, generalmente no se te permite
mutar nada. Piensa en `&` como poner el valor en exhibición en un museo.
`&` es genial para métodos que solo quieren observar `self`.

Más tarde veremos que la regla sobre la mutación puede ser eludida en ciertos
casos. Por eso las referencias compartidas no se llaman referencias *inmutables*.
En realidad, las referencias mutables podrían llamarse referencias *únicas*, pero
hemos descubierto que relacionar la propiedad con la mutabilidad da la intuición
correcta el 99% del tiempo.