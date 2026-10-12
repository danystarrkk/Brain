---
tags:
  - sistemas
  - entorno
  - zsh-suite
tipo: concepto
estado: terminado
---

# Configuración Zsh Batcat, Lsd y Burp Suite

Capa de **herramientas CLI modernas** sobre el shell ya configurado ([[Configuración de Zsh y Plugins]]): `bat` (cat con resaltado), `lsd` (ls con iconos y colores), la cadena `fd`/`ripgrep`/`fzf` para búsqueda, y los fixes de entorno para **Burp Suite** en un WM de mosaico.

## Instalación de Batcat y Lsd

```bash
sudo apt install bat lsd
```

> [!info] `bat` (binario `batcat` en Debian/Ubuntu)
> `bat` es un `cat` con superpoderes: resaltado de sintaxis para decenas de lenguajes, numeración de líneas, marcado de git (líneas añadidas/modificadas), paginación automática con `less` y detección automática de tipos de archivo. Además reconstruye el arte ASCII perdido de las cajas de código.
>
> > [!warning] El gotcha del renombrado en Debian
> > Debian y Ubuntu **renombran el binario a `batcat`** por conflicto con el antiguo `bat` (un gestor de baterías en Perl, paquete `bacula`). En Arch/Gentoo el binario es simplemente `bat`. Por eso los aliases de la sección siguiente apuntan a `batcat`: en Debian `bat` directamente no existe.

> [!info] `lsd` (LSDeluxe)
> Alternativa moderna a `ls`: iconos por tipo de archivo (usa los glifos de [[Configuración Post-Instalación de Entorno de Escritorio|Nerd Fonts]]), colores, árbol opcional (`--tree`), tamaño legible (`-lh`), fechas relativas y agrupación de directorios. Sintaxis compatible con `ls` en sus flags habituales (`-l`, `-a`, `-R`).

## Personalización de Zsh: colores y PATH

```bash
export LS_COLORS="rs=0:di=34:ln=36:mh=00:pi=40;33:so=35:do=35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=00:tw=30;42:ow=34;42:st=37;44:ex=32:*.tar=31:*.tgz=31:*.arc=31:*.arj=31:*.taz=31:*.lha=31:*.lz4=31:*.lzh=31:*.lzma=31:*.tlz=31:*.txz=31:*.tzo=31:*.t7z=31:*.zip=31:*.z=31:*.dz=31:*.gz=31:*.lrz=31:*.lz=31:*.lzo=31:*.xz=31:*.zst=31:*.tzst=31:*.bz2=31:*.bz=31:*.tbz=31:*.tbz2=31:*.tz=31:*.deb=31:*.rpm=31:*.jar=31:*.war=31:*.ear=31:*.sar=31:*.rar=31:*.alz=31:*.ace=31:*.zoo=31:*.cpio=31:*.7z=31:*.rz=31:*.cab=31:*.wim=31:*.swm=31:*.dwm=31:*.esd=31:*.avif=35:*.jpg=35:*.jpeg=35:*.mjpg=35:*.mjpeg=35:*.gif=35:*.bmp=35:*.pbm=35:*.pgm=35:*.ppm=35:*.tga=35:*.xbm=35:*.xpm=35:*.tif=35:*.tiff=35:*.png=35:*.svg=35:*.svgz=35:*.mng=35:*.pcx=35:*.mov=35:*.mpg=35:*.mpeg=35:*.m2v=35:*.mkv=35:*.webm=35:*.webp=35:*.ogm=35:*.mp4=35:*.m4v=35:*.mp4v=35:*.vob=35:*.qt=35:*.nuv=35:*.wmv=35:*.asf=35:*.rm=35:*.rmvb=35:*.flc=35:*.avi=35:*.fli=35:*.flv=35:*.gl=35:*.dl=35:*.xcf=35:*.xwd=35:*.yuv=35:*.cgm=35:*.emf=35:*.ogv=35:*.ogx=35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.m4a=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.oga=00;36:*.opus=00;36:*.spx=00;36:*.xspf=00;36:*~=00;90:*#=00;90:*.bak=00;90:*.old=00;90:*.orig=00;90:*.part=00;90:*.rej=00;90:*.swp=00;90:*.tmp=00;90:*.dpkg-dist=00;90:*.dpkg-old=00;90:*.ucf-dist=00;90:*.ucf-new=00;90:*.ucf-old=00;90:*.rpmnew=00;90:*.rpmorig=00;90:*.rpmsave=00;90:"
export PATH=/opt/kitty/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

> [!info] `LS_COLORS`
> Cadena `clave=valor` separada por `:` que asigna **códigos ANSI a patrones de archivo**: `di=34` (directorio azul), `ex=32` (ejecutable verde), `ln=36` (symlink cian), `su=37;41` (setuid con fondo rojo), `*.tar=31` (comprimidos rojos), `*~=00;90` (backups gris tenue). Consumen esta variable: `ls --color`, **`lsd`**, el completado de zsh (`list-colors`) y `dircolors`. Es el mismo mecanismo que usa polybar para los indicadores, sólo que ahí los colores van en el `.ini`.

> [!info] `PATH` personal
> El orden declara la prioridad de búsqueda de binarios: `/opt/kitty/bin` (kitty instalado fuera de apt), luego las estándar `/usr/local/bin` → `/usr/bin`, y las de juegos al final. El orden importa: un binario duplicado en dos directorios gana el primero. **Advertencia**: sobrescribir `PATH` en `.zshrc` borra los aportes de `.zprofile`/`.zshenv`; lo robusto es `export PATH="/opt/kitty/bin:$PATH"` (preponderancia sin borrar). Se conserva aquí el valor literal del setup personal.

> [!warning] Corrección de formato aplicada
> En la nota original, el callout de `LS_COLORS` y el encabezado "Alias Personalizados" quedaban **dentro** del bloque de código bash abierto (faltaba cerrarlo tras los `export`), por lo que se renderizaban como texto plano. Aquí cada bloque está correctamente delimitado — contenido sin cambios.

## Alias personalizados

```bash
# Custom Aliases
# -----------------------------------------------
# bat
alias cat='batcat'
alias catn='batcat --style=plain'
alias catnp='batcat --style=plain --paging=never'
 
# ls
alias ll='lsd -lh --group-dirs=first'
alias la='lsd -a --group-dirs=first'
alias l='lsd --group-dirs=first'
alias lla='lsd -lha --group-dirs=first'
alias ls='lsd --group-dirs=first'
```

Tabla de la config personal:

| Alias | Expansión | Notas |
|---|---|---|
| `cat` | `batcat` | Todo lo que use `cat` a secas (scripts, `man` con `manpager`) hereda resaltado |
| `catn` | `batcat --style=plain` | Sin bordes/decoración — salida "limpia" para copiar |
| `catnp` | `… --paging=never` | Clave para **uso dentro de scripts/pipes**: sin esto bat paginaría y se colgaría esperando `less` |
| `ll` | `lsd -lh --group-dirs=first` | Lista larga, tamaño legible, directorios agrupados arriba |
| `la` / `lla` | variantes con `-a` | Incluyen archivos ocultos |
| `ls` | `lsd …` | Reemplazo total: cada `ls` es ya `lsd` |

> [!tip] `--group-dirs=first` y `--group-dirs=last`
> `lsd` puede ordenar directorios primero (estilo GNOME) o mezclarlos alfabéticamente. La elección personal es `first`. Nota: alias de `ls` no afectan a los `ls` **dentro de scripts** con `#!/bin/sh` (los alias no se expanden ahí) — ventaja en scripts como los de [[Configuración de Polybar|polybar]]: la salida programática se mantiene estable.
>
> Extras de `lsd` que conviene conocer: `lsd --tree -d 2` (árbol de 2 niveles), `lsd --oneline`, y su propio archivo de config en `~/.config/lsd/config.yaml` (temas y iconos por tipo).

### Config avanzada de `bat`

Los alias cubren el 90 %; para más control, `~/.config/bat/config`:

```bash
--theme="Coldark-Dark"     # cualquier tema de 'batcat --list-themes' o uno instalado a mano
--style="numbers,changes"
--italic-text=always
```

> [!tip] Tema alineado con la terminal
> La paleta de la terminal es Tokyo Night (kitty, ver [[Configuración Post-Instalación de Entorno de Escritorio]]). El tema Tokyo Night de `bat` no viene embebido: se instala descargando el `.tmTheme` a `~/.config/bat/themes/` y ejecutando `batcat --cache-build`. Así bat, `lsd` y el prompt comparten lenguaje cromático.

| Comando | Uso |
|---|---|
| `batcat --list-themes` | Temas disponibles (preview con `batcat --list-themes | fzf`) |
| `batcat -l python -` | Resaltado forzado leyendo de stdin |
| `batcat --diff a b` | Differ con resaltado — sustituye a `diff` + editor para revisiones rápidas |
| `batcat man` / `MANPAGER=\"batcat -l man -p\" man ls` | Páginas de man con resaltado (setup clásico) |
| `batcat --wrap=character` | Ajuste de línea para logs largos |

### Config avanzada de `lsd`

`~/.config/lsd/config.yaml` — la contraparte del `.zshrc`:

```yaml
color:
  when: auto
layout: grid          # o 'oneline' / 'tree'
icons:
  when: always         # requiere Nerd Font en la terminal
  theme: fancy
date: relative         # "hace 2 horas" en vez de timestamp
group-dirs: first      # lo que los aliases ya fuerzan; útil para invocar 'lsd' sin alias
```

> [!info] Aliases vs config
> Las opciones de `config.yaml` aplican a **cada** invocación de `lsd`; las de los aliases solo a las que pasan por ellos. `group-dirs: first` en el YAML hace redundante el flag del alias — dejarlo en uno u otro según preferencia de mantener la config en un solo sitio.

## Cadena de búsqueda moderna: fd, ripgrep, fzf

Complementos naturales de este setup (el propio flujo del entorno los usa):

```bash
sudo apt install fd-find ripgrep fzf
```

| Herramienta | Reemplaza | Uso canónico | Gotcha Debian |
|---|---|---|---|
| `fd` | `find` | `fd 'patrón' src/` — regex, ignora `.gitignore`, color, rápido | El binario se llama **`fdfind`** en Debian (mismo conflicto de nombres); alias: `alias fd=fdfind` |
| `ripgrep` | `grep -r` | `rg 'patrón'` — recursivo, respeta `.gitignore`, literal por defecto | binario `rg` normal |
| `fzf` | — | Fuzzy-finder interactivo: `Ctrl+R` (historial), `Ctrl+T` (archivos), `Alt+C` (cd) | `fzf --zsh` genera los keybindings para zsh |

```bash
# integración típica en ~/.zshrc
source <(fzf --zsh)                       # keybindings + completado fzf
export FZF_DEFAULT_OPTS='--height 40% --reverse --border'
export FZF_DEFAULT_COMMAND='fd --type f'  # qué lista Ctrl+T
```

> [!tip] Flujo completo
> `rg` encuentra el patrón en el código → `fzf` selecciona interactivo → `batcat` lo muestra resaltado: `rg -l 'nologin' | fzf | xargs batcat`. Es el "superpoder" del这套 tooling — tres binarios pequeños que sustituyen a `find | grep | less` con mejor rendimiento y UX. Ver también [[Comandos Básicos de Sistema y Red en Linux]] para los comandos clásicos que estos modernizan.

## Solución de problemas para Burp Suite

### Lanzamiento roto tras actualizar el sistema

Burp Suite puede dejar de lanzarse tras upgrades del sistema (conflictos de runtime Java empaquetado). La solución documentada:

[Ver solución para Burp Suite (Medium)](https://medium.com/@neat_mahogany_porcupine_191/burpsuite-no-longer-launches-after-parrot-upgrade-d1c6b17cb70d)

### Interfaz desproporcionada en WM de mosaico (fix #1)

Resuelto el lanzamiento, la UI puede mostrarse escalada/mal posicionada. Fix en el `.zshrc`:

```bash
# Fix the Java Problem
export _JAVA_AWT_WM_NONREPARENTING=1
```

> [!tip] `_JAVA_AWT_WM_NONREPARENTING=1`
> Java AWT asume por defecto que el WM practica **reparenting** (las ventanas se insertan dentro de una ventana-marco gestionada por el WM). bspwm/i3/dwm **no** reparentean, y Java malinterpreta la geometría: ventanas diminutas, gigantes o mal centradas. La variable le dice a AWT que detecte el entorno "sin reparenting" y calcule la decoración correctamente. Sin ella, Burp (y cualquier app Java/Swing) se dibuja mal en este entorno.

### Gestión del WM reportado a Java (fix #2)

Además, en la configuración de bspwm (el `bspwmrc`, ver [[Configuración de BSPWM y SXHKD]]):

```bash
wmname LG3D &
```

> [!info] `wmname LG3D`
> Algunas apps Java (Burp incluido) contienen *workarounds* específicos por WM y detectan el nombre del gestor en la raíz X11. `wmname` cambia ese nombre; **`LG3D`** es el nombre que Java 3D usaba — al reportarlo, las apps Java activan su ruta de compatibilidad y dejan de intentar ajustes rotos. `&` por ser un comando de una sola ejecución que no debe bloquear `bspwmrc`. Es el mismo truco documentado en [[Configuración Post-Instalación de Entorno de Escritorio]] y en el arranque del entorno.

### Ajustes de JVM y flujo de trabajo (referencia)

Los settings personales del entorno anteriores + los ajustes recomendados de Burp:

| Ajuste | Dónde | Valor recomendado |
|---|---|---|
| Memoria JVM | *Project options → JVM* o `~/.BurpSuite/.vmoptions`: `-Xmx` | Labos grandes: `-Xmx4g` (máquina con 8 GB+); default 2 GB se queda corto con escaneos |
| Proxy | *Proxy → Options → Listeners*: `127.0.0.1:8080` | Escucha solo en loopback; el navegador (con FoxyProxy) apunta al mismo puerto y **HTTPS proxy** con certificado CA de Burp instalado |
| Proyecto | *Burp → Settings → Project* | **Temporary projects** para labos rápidos (todo en RAM); **project files** `.burp` persistentes cuando hay que retomar o compartir |
| Ficheros de config | `~/.BurpSuite/` | Burp guarda config de UI/proyectos aquí; respaldar junto a los dotfiles |

### Flujo de proxy completo (navegador ↔ Burp)

1. Burp escucha en `127.0.0.1:8080` (listener por defecto).
2. Navegador: proxy manual `127.0.0.1:8080` para HTTP y HTTPS (FoxyProxy facilita el on/off).
3. **Certificado CA**: navegar a `http://burp` en la sesión proxy → *CA Certificate* → importarlo en el navegador (Ajustes → Privacidade y seguridad → Certificados → Autoridades). Sin esto, todo HTTPS aparece como error de certificado.
4. Fuera del laboratorio: **desactivar el proxy** (FoxyProxy off) — un proxy apagado con el navegador apuntando a él corta la red por completo.

> [!info] `-Xmx` y el resto de flags JVM
> `-Xmx` fija la memoria **máxima** del heap; `-Xms` la inicial. Burp expone `-Xmx` en su diálogo de settings, pero el archivo `.vmoptions` (junto al binario o en `~/.BurpSuite/`) permite flags adicionales (`-Djava.awt.headless=false` en casos raros). Al cambiar `-Xmx` la JVM nueva debe relanzarse — Burp muestra el heap efectivo en *Help → Memory*.

> [!warning] Después de cambiar entorno
> Los exports y el `wmname` requieren **cerrar sesión y volver a iniciar** (o al menos abrir una nueva sesión de login para que `.zshrc`/`bspwmrc` se re-ejecuten) para que las apps Java nuevas arranquen con el entorno corregido.

## Errores comunes

1. **`bat: command not found` en Debian** → usar `batcat` (o el alias `cat='batcat'`).
2. **`fd: command not found` en Debian** → el binario es `fdfind`.
3. **`batcat` dentro de un script se cuelga** → falta `--paging=never` (`catnp`), `less` espera input.
4. **Iconos de `lsd` como □** → fuentes Nerd ausentes (paso 2 de [[Configuración Post-Instalación de Entorno de Escritorio]]).
5. **Burp sigue tiny tras los fixes** → verificar que la shell que lanza Burp exporta `_JAVA_AWT_WM_NONREPARENTING` (no vale con tenerlo en otra sesión) y que `wmname` corrió en la sesión actual (`xprop | grep WM_NAME`).
6. **`LS_COLORS` no surte efecto** → la variable debe exportarse **antes** de lanzar la app; para `ls` de coreutils hace falta `--color=auto`/alias.

## Cheatsheet

| Comando | Acción |
|---|---|
| `batcat archivo` / `cat archivo` | Ver con resaltado |
| `batcat -l python` | Forzar lenguaje |
| `batcat --list-themes` / `-t tema` | Temas de color |
| `lsd -lah --tree` | Lista con árbol |
| `fdfind '\.conf$'` | Buscar archivos |
| `rg -n 'patrón'` | Buscar en contenido |
| `fzf` / `Ctrl+R` / `Ctrl+T` | Fuzzy find / historial / archivos |
| `xprop \| grep WM_CLASS` | Clase X11 de una ventana (para exclusiones picom/rules) |

## Recursos

- bat: <https://github.com/sharkdp/bat>
- lsd: <https://github.com/lsd-rs/lsd>
- fd: <https://github.com/sharkdp/fd>
- ripgrep: <https://github.com/BurntSushi/ripgrep>
- fzf: <https://github.com/junegunn/fzf>
- Burp Suite: <https://portswigger.net/burp/documentation/desktop>

## Relacionadas

- [[Configuración de Zsh y Plugins]] — el `.zshrc` donde viven estos exports y aliases
- [[Configuración Post-Instalación de Entorno de Escritorio]] — Nerd Fonts (iconos de lsd/bat) y kitty
- [[Configuración de Polybar]] — scripts que consumen el entorno modernizado
- [[Configuración de BSPWM y SXHKD]] — `wmname` y el arranque de Burp en bspwm
- [[Comandos Básicos de Sistema y Red en Linux]] — las versiones clásicas que estas herramientas modernizan
