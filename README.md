# Aprende Rust escribiendo Demasiadas Listas Enlazadas
[![Build Status](https://github.com/rust-unofficial/too-many-lists/actions/workflows/main.yml/badge.svg)](https://github.com/rust-unofficial/too-many-lists/actions)

Lee la versión bonita en https://rust-unofficial.github.io/too-many-lists/.

# Construcción (Build)

Para construir se requiere mdbook, que se puede instalar desde crates.io:

```sh
cargo install mdbook
```

Asumiendo que has puesto el directorio de instalación `~/.cargo/bin` en tu PATH del sistema, entonces ejecuta desde la raíz de tu copia local:

```sh
mdbook build
```

---

Si lo prefieres, este proyecto también se puede construir con
[GitBook](https://github.com/GitbookIO/gitbook), aunque GitBook
no tiene soporte oficial y la compatibilidad es por lo tanto
incierta e incidental.