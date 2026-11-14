# New (Nuevo)

Para asociar código real con un tipo, usamos bloques `impl`:

```rust ,ignore
impl List {
    // TODO, hacer que el código ocurra
}
```

Ahora solo necesitamos averiguar cómo escribir código realmente. En Rust declaramos
una función así:

```rust ,ignore
fn foo(arg1: Type1, arg2: Type2) -> ReturnType {
    // body
}
```

Lo primero que queremos es una forma de *construir* una lista. Como ocultamos los
detalles de implementación, necesitamos proporcionar eso como una función. La forma
habitual de hacerlo en Rust es proporcionar un método estático, que es solo una
función normal dentro de un `impl`:

```rust ,ignore
impl List {
    pub fn new() -> Self {
        List { head: Link::Empty }
    }
}
```

Algunas notas sobre esto:

* Self es un alias para "ese tipo que escribí arriba junto a `impl`". ¡Genial para
  no repetirte!
* Creamos una instancia de un struct de la misma manera que lo declaramos, excepto
  que en lugar de proporcionar los tipos de sus campos, los inicializamos con valores.
* Nos referimos a las variantes de un enum usando `::`, que es el operador de
  namespacing.
* La última expresión de una función se retorna implícitamente.
  Esto hace que las funciones simples sean un poco más ordenadas. Aún puedes usar
  `return` para retornar anticipadamente como en otros lenguajes tipo C.