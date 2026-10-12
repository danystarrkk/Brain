---
aliases:
  - Vim
  - nvim
  - Editor de texto Vim
tags:
  - sys/entorno
  - linux/comandos
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Configuración Post-Instalación de Entorno de Escritorio]]"
  - "[[Configuración de BSPWM y SXHKD]]"
  - "[[Bash Scripting - Fundamentos Básicos]]"
fuente: "https://vim.rtorr.com/"
---

# Vim: Comandos Esenciales para la Edición Rápida

**Vim** es un editor modal: no escribe mientras navega — alterna entre **modos**, y en modo normal cada tecla es un *comando* compuesto por **operador + movimiento** (`d` + `w` = borrar palabra). Entender esa gramática es entender Vim: en vez de memorizar cientos de comandos, se aprenden 3 operadores y 10 movimientos, y sus combinaciones son la multiplicación. Navegar sin el ratón acelera drásticamente la edición de los configs de todo el entorno ([[Configuración de BSPWM y SXHKD]], [[Configuración de Polybar]], scripts — ver [[Bash Scripting - Fundamentos Básicos]]).

> [!info] Vim vs Neovim
> **Neovim** (`nvim`) es un fork moderno de Vim: mismo modelo modal y compatible con casi toda su sintaxis, pero con Lua como lenguaje de config (init.lua), LSP integrado, plugins async y comunidad actual. Esta nota aplica a ambos: los comandos de modo normal son idénticos; la config (ver "init.vim / vimrc") cambia de formato. En Debian: `sudo apt install vim` / `sudo apt install neovim`.

## Los modos

| Modo | Entrar desde | Salir | Para qué |
|---|---|---|---|
| **Normal** | `Esc` desde cualquier modo | — | Navegar y ejecutar comandos. Es el modo "casa" |
| **Insertar** | `i`, `a`, `o`, `I`, `A`, `O`… | `Esc` (o `jk` en la config personal) | Escribir texto |
| **Visual** | `v` (caracteres), `V` (línea), `Ctrl+v` (bloque/columna) | `Esc` | Seleccionar y operar sobre la selección |
| **Comandos** | `:` | `Enter` | Ex commands: guardar, buscar/replace, abrir archivos |
| **Replace** | `R` | `Esc` | Sobrescribir carácter a carácter |
| **Terminal-Job** (nvim) | `:terminal` | `Ctrl+\ Ctrl+n` | Shell embebido |

La clave del modelo: en Insert **solo escribes**; en Normal **solo comandos**. La regla de oro — si algo "no hace nada raro", casi seguro estás en el modo equivocado (mirar la esquina inferior izquierda).

> [!tip] Salir de Insert: `Esc` o `jk`
> El método estándar es `Esc`. La config personal de este vault mapea **`jk`** a salir del modo insert (mapping típico en `.vimrc`: `inoremap jk <Esc>`, que además evita el viaje a la tecla Esc). Nota: con `jk` mapeado, teclear "jk" literal dentro de palabras es imposible en insert — `Ctrl+v j` escribe el carácter sin triggers.

## Comandos esenciales (la tabla base del vault)

| Comandos | Descripción |
| --- | --- |
| `i` | Entra en modo inserción; el cursor se posiciona **a la izquierda** del carácter actual |
| `a` | Entra en modo inserción; el cursor se posiciona **a la derecha** del carácter actual |
| `h, j, k, l` | Navegación en modo normal: `h` izquierda, `j` abajo, `k` arriba, `l` derecha |
| `jk` | Sale del modo inserción y vuelve al modo normal (mapping personal) |
| `0` | Mueve el cursor al principio de la línea |
| `$` | Mueve el cursor al final de la línea |
| `w` | Mueve el cursor una palabra hacia la derecha |
| `d` | Operador de borrado: su comportamiento depende del movimiento (`dw` borra una palabra, `d$` hasta el final de la línea) |
| `dd` | Borra la línea completa (el operador `d` aplicado a sí mismo = `d d`) |
| `v` | Modo visual: selecciona texto para copiar/cortar/pegar |
| `o` | Crea una nueva línea **debajo** de la actual y entra en modo inserción (`O` = arriba) |
| `q + a` | **Graba** una macro en el registro `a`; `q` de nuevo detiene la grabación |
| `@<registro>` | **Ejecuta** la macro del registro (`@a` repite la grabada, `@@` la última) |
| `/<valor>` | Búsqueda hacia adelante; `n` = siguiente coincidencia, `N` = anterior |
| `:` | Línea de comandos. Ejemplo personal: `:%s/nologin/yeslogin/g` sustituye en **todo el archivo** (`%` = rango archivo, `s/` = substitute, `g` = todas las ocurrencias por línea) |

## Gramática: operadores + movimientos

La potencia de Vim es componer. Un comando = **[contador] operador [contador] movimiento**:

| Operador | Letra | `dw` | `d$` | `dG` |
|---|---|---|---|---|
| Borrar | `d` | borra la palabra | hasta fin de línea | hasta el final del archivo |
| Copiar (yank) | `y` | yank de la palabra | yank a fin de línea | yank hasta EOF |
| Cambiar (borra y entra en insert) | `c` | cambia la palabra | cambia a fin de línea | — |
| Indentar | `>` / `<` | indenta/desindenta | — | — |

Movimientos (actúan como "hasta dónde"):

| Movimiento | Alcance |
|---|---|
| `w` / `b` | Siguiente / anterior **palabra** (por puntuación la separa) |
| `e` | Fin de palabra |
| `0` / `^` / `$` | Inicio de línea / primer carácter / fin de línea |
| `(`/`)` | Paréntesis |
| `{` / `}` | Bloque en blanco (párrafos) |
| `gg` / `G` | Inicio / final del archivo; `42G` = línea 42 |
| `%` | Complemento del delimitador bajo el cursor `(){}[]` |

Con **contador**: `3dw` borra 3 palabras; `d3w` idem; `5j` baja 5 líneas; `2dd` borra 2 líneas.

> [!tip] El comando que lo cambia todo: `ci"`
> `ci"` = **c**hange **i**nside las comillas: borra el texto *dentro* de las comillas y entra en insert. Funciona igual con `ci{}`, `ci)`, `ci'`. Para copiar en vez de cambiar: `yi"`. En configs ([[Configuración de Polybar]], `.zshrc`) es el comando más usado de Vim.

## Edición de texto

| Comando | Acción |
|---|---|
| `x` / `X` | Borrar carácter bajo / anterior |
| `D` / `C` | Borrar / cambiar hasta fin de línea (`d$`/`c$`) |
| `s` / `S` | Sustituir carácter / línea completa |
| `p` / `P` | Pegar debajo / arriba del cursor (registro o clipboard) |
| `u` / `Ctrl+r` | Deshacer / rehacer |
| `.` | **Repite el último cambio** — el "atajo definitivo" |
| `J` | Une la línea actual con la siguiente |
| `>>` / `<<` | Indenta / desindenta la línea |
| `r<char>` | Reemplaza el carácter bajo el cursor (uno) |
| `~` | Invierte mayúsculas del carácter |

### Mecánica de `. ` (dot-repeat)

`.` repite el **último cambio completo** como un "macro de una tecla": borrar una palabra (`dw`), mover con `w` y pulsar `.` tres veces borra las tres siguientes. La técnica experta: **editar la primera ocurrencia de forma consciente** (por ejemplo `ciw<texto>Esc`) y dejar que `.` replique el cambio en el resto mientras navegas con `/patrón` + `n`. Vim guarda el cambio como operación atómica — insert incluido — por eso `.` es la base de la productividad.

## Registros y yank

Todo lo que Vim "corta/copia" va a **registros** (no a un portapapeles único):

| Registro | Contenido |
|---|---|
| `"` (unnamed) | El último yank/borrado — lo que `p` pega por defecto |
| `a–z` | Registros con nombre: `"ayy` yanka la línea en `a`, `"ap` la pega |
| `0` | El **último yank** (no lo pisan los borrados — clave para borrar y luego pegar lo yankado) |
| `1–9` | Los últimos borrados, más antiguos → más recientes |
| `+` / `*` | Clipboard del sistema (X11/Wayland): `"+y`, `"+p` |

El ejemplo de sustitución personal es también un yank implícito: cada `s/` deja lo borrado en el registro `1`.

## Macros: automatización real

Flujo completo (la sección `q+a`/`@a` de la tabla base, expandida):

1. `qa` — empieza a grabar todo en el registro `a` (la barra de estado muestra "recording @a").
2. Ejecutar la secuencia: por ejemplo `I# <Esc>j` (comentar la línea y bajar).
3. `q` — detiene la grabación.
4. `@a` — repite; `10@a` — repite diez veces; `@@` — repite la última macro.

> [!info] Macro sobre un rango
> El patrón maestro: seleccionar en visual las líneas, `:'<,'>normal @a` ejecuta la macro `a` en cada línea seleccionada. Alternativa sin seleccionar: `:10,20norm @a`. Las macros son "solo" normal-mode commands grabados — cualquier cosa grabable es reproducible.

## Búsqueda y sustitución

| Comando | Efecto |
|---|---|
| `/patrón` | Busca hacia adelante (regex Vim); `?patrón` hacia atrás |
| `n` / `N` | Siguiente / anterior coincidencia |
| `\*` | Busca la palabra **bajo el cursor** |
| `*` / `#` | Ídem como wildcard |
| `:%s/old/new/g` | Todo el archivo, todas por línea (el ejemplo personal de `nologin`→`yeslogin`) |
| `:%s/old/new/gc` | Igual pero **confirma** cada sustitución (y/n/a/l/q) — recomendado en configs |
| `:'<,'>s/old/new/g` | Solo en el rango visual seleccionado |
| `:%s/\v[0-9]+/N/g` | `\v` = "very magic": regex más cercana a lo estándar |

Flags: `g` global por línea, `c` confirmar, `i` ignorar mayúsculas, `n` contar sin reemplazar. Búsquedas con `\v` + `[0-9]{1,3}` hacen los patrones legibles.

## Ventanas (splits), pestañas y buffers

| Comando | Acción |
|---|---|
| `:sp` / `:vs` | Split horizontal / vertical del buffer actual |
| `Ctrl+w h/j/k/l` | Moverse entre splits (como las flechas de bspwm) |
| `Ctrl+w _` / `Ctrl+w \|` | Maximizar alto / ancho del split |
| `:tabe archivo` | Abrir en pestaña; `gt`/`gT` cambiar de pestaña |
| `:e archivo` | Abrir en el buffer actual |
| `:ls` | Listar buffers; `:bn`/`:bp` siguiente/anterior |
| `:q` / `:qa` / `:wq` / `ZZ` | Cerrar / cerrar todo / guardar+y-cerrar / atajo |

> [!tip] Vim en un WM de mosaic
> Los splits de Vim conviven con los paneles de bspwm — la filosofía coincide (navegar con teclado, sin ratón). Los `Ctrl+w h/j/k/l` son al `super + flecha` de [[Configuración de BSPWM y SXHKD]] lo que los atajos de kitty son a los splits del terminal.

## `init.vim` / `vimrc`: configuración básica

| | Vim | Neovim |
|---|---|---|
| Archivo | `~/.vimrc` (+ `~/.vim/`) | `~/.config/nvim/init.vim` o `init.lua` |
| Sistema de plugins | vundle / vim-plug / pathogen | lazy.nvim / packer (Lua) |

Config mínima (aplica a ambos; `init.vim` en neovim):

```vim
set number          " números de línea
set relativenumber  " números relativos (para operadores con contador)
set tabstop=4 shiftwidth=4 expandtab   " tabs → 4 espacios
set ignorecase smartcase   " búsqueda insensible, salvo con mayúsculas
set incsearch hlsearch     " búsqueda incremental y resaltado
set clipboard=unnamedplus  " yank/paste con el clipboard del sistema
syntax on
filetype plugin indent on
```

El repositorio de dotfiles personal guarda aquí sus mappings (`inoremap jk <Esc>`) y las preferencias que heredan todas las sesiones — editar con los mismos comandos de esta nota.

## Recetas prácticas

### 1. Comentar/descomentar un bloque de config

Selección visual + norm (el atajo de operador sobre rango):

```
vip             " selecciona el párrafo (inner paragraph)
:'<,'>norm #    " antepone '#' a cada línea
```

Para descomentar: `:'<,'>norm 0x` (borra el primer carácter de cada línea — cuidado con la indentación; variante robusta: `:'<,'>s/^# \?//`). En un `.ini` de [[Configuración de Polybar]] o un `bspwmrc` esto se usa constantemente para probar/desactivar módulos.

### 2. Sustitución consciente en configs con valores personales

Antes de tocar valores sensibles (paths de `/home/stark/...`, interfaces `wlp2s0`, `BAT1`):

```
:%s/wlp2s0/wlp3s0/gc    " con 'c' confirma cada reemplazo — seguro para dotfiles
:%s/\v\/home\/stark/\/home\/usuario/g   " mover la config a otra máquina, very-magic
```

Patrón recomendado del vault: `%` + `s` + flag `c` — ver la sección de búsqueda/sustitución arriba.

### 3. Macro de mantenimiento masivo

Ejemplo real: aplicar el mismo cambio a 40 scripts de `~/.config/bspwm/scripts/`:

```
:args *.sh              " carga todos los scripts como argumentos
qa                       " graba: Iset -eu<Esc>q (p. ej. añadir cabecera)
:argdo normal @a | update   " ejecuta la macro en cada archivo y guarda
```

`| update` guarda solo si hubo cambios — evita tocar mtimes sin motivo.

### 4. Auditar configs sin abrir una por una

```
:grep TODO ~/.config/**/*.ini   " o :vimgrep — lista coincidencias en el quickfix
:copen                          " ventana de resultados; :cnext/:cprev navega
```

En neovim, `:grep` delega en `rg` si está instalado (ver [[Configuración Zsh Batcat Lsd y Burp Suite]]), respetando `.gitignore`.

## Errores comunes

1. **"No puedo escribir"** → estás en modo Normal: `i` para insertar. Mirar la esquina inferior izquierda.
2. **No hay salida de Vim** (`:q!` no reconocido) → pulsar `Esc` primero; `:` abre comandos solo desde Normal.
3. **`jk` escribe las letras en vez de salir** → el mapping no está cargado (falta en vimrc) o estás en modo Replace (`R`).
4. **`p` pega texto viejo** → lo que cortaste con `x`/`d` está en el registro `1`, no en `"`; yank con `y` explícito, o usar `"_d` para borrar sin tocar el registro.
5. **Sustitución masiva a medias** → faltó la `g` final (`s/` solo reemplaza la primera aparición **por línea**) — o añadir `c` para confirmar.
6. **Cursor en modo "cuadrado" tras salir** → es normal en TTY; en terminales modernos se cambia el shape desde la config del propio terminal (kitty: `cursor_shape`, ver [[Configuración Post-Instalación de Entorno de Escritorio]]).

## Cheatsheet de bolsillo

| Tecla | Acción | Tecla | Acción |
|---|---|---|---|
| `i` / `a` / `o` | Insert antes / después / línea nueva | `dd` / `yy` / `p` | Borrar línea / yank / pegar |
| `h j k l` | Flechas | `dw` / `cw` | Borrar / cambiar palabra |
| `w b e` | Palabra adelante / atrás / fin | `ci"` / `yi(` | Cambiar / yank inside |
| `0 $` | Inicio / fin de línea | `.` | Repetir cambio |
| `gg G` | Inicio / fin archivo | `u` / `Ctrl+r` | Undo / redo |
| `/pat` `n N` | Buscar, siguiente, anterior | `qa … q` `@a` | Grabar / ejecutar macro |
| `v` / `V` / `Ctrl+v` | Visual char / línea / bloque | `:%s/a/b/gc` | Replace confirmado |
| `:w` `:q` `:wq` | Guardar / salir / guardar-salir | `Ctrl+w s/v` | Splits |

Referencia extensa interactiva (la usada al aprender): <https://vim.rtorr.com/> — y `vimtutor` instalado con el propio vim: el tutorial de 30 minutos definitivo.

## Relacionadas

- [[Configuración Post-Instalación de Entorno de Escritorio]] — el editor con el que se escriben los configs del entorno
- [[Configuración de BSPWM y SXHKD]] — filosofía de navegación por teclado compartida
- [[Configuración de Polybar]] — `.ini`s editados con estos comandos
- [[Bash Scripting - Fundamentos Básicos]] — scripts donde aplican macros y sustituciones
- [[Comandos Básicos de Sistema y Red en Linux]] — el resto de la línea de comandos
