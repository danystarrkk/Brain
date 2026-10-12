---
tags:
  - sistemas
  - entorno
  - zsh-plugins
tipo: concepto
estado: terminado
---

# Configuración de Zsh y Plugins

**Zsh** (*Z Shell*) es el shell interactivo de esta pila: corre dentro de kitty (ver [[Configuración Post-Instalación de Entorno de Escritorio]]), lanza las apps desde sxhkd y alimenta los scripts de [[Configuración de Polybar]]. Esta nota documenta el setup real: plugins de Debian cargados a mano (sin framework), prompt **Powerlevel10k**, historial compartido con root por enlace simbólico y el sistema de completado (`zstyle`) configurado al detalle.

> [!info] Por qué zsh y no bash
> Zsh es retrocompatible con bash en el día a día pero añade: **globbing** rico (`**/*.txt` recursivo), **sustitución de expansión** (`=cmd` → `/usr/bin/cmd`), **completado interactivo** con menús seleccionables (zle), corrección ortográfica (`correct`) y el ecosistema de plugins más pulido. Debian/Ubuntu lo usan por defecto desde 2017+ en instalaciones nuevas de escritorio.

## Orden de arranque: por qué el orden de tu `.zshrc` importa

Zsh lee sus archivos de inicio **en un orden fijo y distinto para login/interactivos**:

| Archivo | Cuándo se lee | Qué va ahí |
|---|---|---|
| `/etc/zsh/zshenv` + `~/.zshenv` | **Siempre** (todos los zsh, incluso scripts) | `PATH`, variables globales. Es el único que corre hasta en `zsh -c` |
| `/etc/zsh/zprofile` + `~/.zprofile` | Sesiones de **login** (kitty con `shell zsh`, consola TTY, `zsh -l`) | `export`, `dircolors`, arranque de keychain — lo "de entorno" |
| `/etc/zsh/zshrc` + `~/.zshrc` | Sesiones **interactivas** | Prompt, plugins, aliases, completado — *casi todo lo de esta nota* |
| `/etc/zsh/zlogin` + `~/.zlogin` | Solo login, **después** de `zshrc` | Mensajes de bienvenida, `neofetch` |
| `~/.zlogout` / `/etc/zsh/zlogout` | Al cerrar sesión de login | Limpieza |

Consecuencias prácticas:

- kitty abre una shell **interactiva de login** (por eso los exports del `.zshrc` también funcionan allí, pero conviene saber que un `.zprofile` corre antes).
- Un script que lance `zsh script.sh` **no** lee `.zshrc` — todo lo que un script espere (`PATH` ampliado) debe vivir en `.zshenv`.
- **El orden dentro de `.zshrc` también es semántico**: los `zstyle` deben ir *antes* de `compinit`; el prompt (p10k) define cómo se pintan los segments y los plugins "últimos" de la cadena se dibujan encima.

## Instalación de los plugins

### Paquetes de Debian

```bash
sudo apt install zsh-autosuggestions zsh-syntax-highlighting
```

> [!info] Qué hacen
> - **`zsh-autosuggestions`**: sugiere comandos mientras escribes, en gris, basándose en el **historial** (coincidencia del prefijo). `→` (flecha derecha) o `End` acepta la sugerencia completa; `ctrl + →` acepta solo la siguiente palabra. El orden de aparición sigue la frecuencia/recencia del historial.
> - **`zsh-syntax-highlighting`**: resalta en **verde** los comandos válidos (existentes en el `PATH` o builtins), **rojo** los inválidos y colorea argumentos/globs. Es un *zle hook* que re-pinta la línea tras cada tecla — por eso es el plugin **más caro** y el último en cargarse (ver advertencia más abajo).
>
> Ambos se instalan como archivos en `/usr/share/zsh-*` y se cargan con `source` (el paquete Debian no auto-registra nada).

### Powerlevel10k

[Powerlevel10k (p10k)](https://github.com/romkatv/powerlevel10k) es el tema/prompt: instant-prompt (snapshot del prompt al arrancar, sin esperar a los plugins), segments de git, tiempo de ejecución del último comando, estado de sudo, etc. Se instala siguiendo su README (recomendado: `git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/powerlevel10k` o en `~/.local/share/powerlevel10k` + `source` en el `.zshrc`), y se configura con:

```bash
p10k configure
```

> [!tip] `p10k configure` y el instant prompt
> El asistente genera `~/.p10k.zsh`. La config personal sigue el patrón clásico: `source ~/.p10k.zsh` **inmediatamente después** de sourcear p10k. El *instant prompt* escribe el prompt guardado antes de cargar todo el `.zshrc` — si un plugin imprime algo al inicio, p10k avisa (podcast: "console output generated during instant prompt") y hay que poner ese comando tras `typeset -g POWERLEVEL9K_INSTANT_PROMPT=off` o moverlo a `.zshrc` tardío.
>
> Alternativa moderna: **starship** (`curl -sS https://starship.rs/install.sh | sh` + `eval "$(starship init zsh)"`) — prompt en Rust, config en TOML, comparable en velocidad. p10k es más profundo en segments; starship en simplicidad. En este setup domina p10k.

> [!warning] zsh-sudo
> El plugin `zsh-sudo` (paquete Debian: `sudo apt install zsh-sudo`) añade `Esc Esc` para anteponer `sudo` al comando actual — salva constantemente en labores de root. Se sourcea igual que los otros dos desde la config personal.

## Sincronizar root con el usuario: el enlace simbólico

Configurada la sesión para `stark`, el usuario `root` hereda **el mismo** `.zshrc` mediante enlace simbólico, ejecutado desde el directorio de root:

```bash
ln -s -f /home/stark/.zshrc .zshrc
```

| Fragmento | Significado |
|---|---|
| `ln -s` | Enlace **simbólico** (puntero al archivo, no copia): cambios en el original se reflejan al instante en root |
| `-f` | Fuerza: si `/root/.zshrc` ya existe, lo reemplaza |
| ruta destino | `/home/stark/.zshrc` — la config del usuario |
| sin ruta destino | Crea el enlace con el mismo nombre en el CWD (`/root/`) |

> [!warning] Dependencia bidireccional
> El enlace hace que root dependa de que `/home/stark/.zshrc` exista siempre y de que el usuario sea legible por root (lo es). Si se migra la config a un dotfiles repo, actualizar la ruta del enlace. Para **deshacer**: `rm /root/.zshrc` (borra solo el enlace, no el original) y recrear un `.zshrc` propio de root.

## La configuración personal del `.zshrc`

Los bloques siguientes son la configuración real. Van **después** del bloque de p10k.

### Carga condicional de plugins

```bash
# ZSH Autosuggestions Plugin
# Carga el plugin de autosugerencias si el archivo existe.
if [ -f /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh ];then
    source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
fi

# ZSH Syntax Highlighting Plugin
# Carga el plugin de resaltado de sintaxis si el archivo existe.
if [ -f /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh ];then
    source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
fi

# ZSH Sudo Plugin
# Carga el plugin zsh-sudo si el archivo existe.
# Este plugin permite añadir 'sudo' al inicio del comando actual presionando 'Esc' dos veces.
if [ -f /usr/share/zsh-sudo/sudo.plugin.zsh ];then
    source /usr/share/zsh-sudo/sudo.plugin.zsh
fi
```

- El guard `if [ -f ... ]` hace la config **portátil**: en una máquina sin los paquetes, el shell arranca igual, solo sin ese plugin.
- **Orden**: autosuggestions → syntax-highlighting → sudo. Regla general del ecosistema: `zsh-syntax-highlighting` debe ser el **último** plugin de la lista — los plugins posteriores no quedarían resaltados por no "verlos" sus hooks.

> [!warning] Orden de carga de los plugins
> El resaltador re-pinta la línea completa del prompt; si se carga antes de otros widgets zle, pinta estados viejos. La recomendación oficial del proyecto es literal: *"zsh-syntax-highlighting should be sourced at the end of the zshrc file"*.

### Historial

```bash
# Configuración del Historial de Comandos
# Define el archivo de historial, el tamaño y las opciones de comportamiento.
HISTFILE=~/zsh_history
HISTSIZE=10000
SAVEHIST=10000 # Corregido: SAVEHIST es el que limita cuántas entradas se guardan en el archivo.
setopt histignorealldups sharehistory
```

| Variable / setopt | Significado |
|---|---|
| `HISTFILE=~/zsh_history` | Ruta del archivo de historial (en este setup, `~/zsh_history`; la convención habitual es `~/.zsh_history` — el valor real es el de esta config) |
| `HISTSIZE=10000` | Entradas **en memoria** por sesión |
| `SAVEHIST=10000` | Entradas que se **escriben a disco** al salir — debe ser ≤ `HISTSIZE` |
| `histignorealldups` | Un comando recién añandido elimina la entrada duplicada anterior — `ls`, `ls`, `cd foo`, `ls` deja un solo `ls` |
| `sharehistory` | Las sesiones abiertas comparten el historial en vivo: cada sesión relee el archivo y las nuevas entradas son visibles en las demás (`inc_append_history` combinable para escritura inmediata por comando) |

> [!info] Corrección aplicada
> La config original traía `HISTLIST=10000`, que **no es una variable de zsh** para este fin (no existe `HISTLIST` estándar). La correcta es `SAVEHIST`, ya usada aquí. `fc -l`, `history 50` y `!patter:n` navegan el historial; `Ctrl+R` es el buscador incremental (mejorable con `fzf`, ver [[Configuración Zsh Batcat Lsd y Burp Suite]]).

### Sistema de completado

```bash
# Use modern completion system
autoload -Uz compinit
compinit

zstyle ':completion:*' auto-description 'specify: %d'
zstyle ':completion:*' completer _expand _complete _correct _approximate
zstyle ':completion:*' format 'Completing %d'
zstyle ':completion:*' group-name ''
zstyle ':completion:*' menu select=2
eval "$(dircolors -b)"
zstyle ':completion:*:default' list-colors ${(s.:.)LS_COLORS}
zstyle ':completion:*' list-colors ''
zstyle ':completion:*' list-prompt %SAt %p: Hit TAB for more, or the character to insert%s
zstyle ':completion:*' matcher-list '' 'm:{a-z}={A-Z}' 'm:{a-zA-Z}={A-Za-z}' 'r:|[._-]=* r:|=* l:|=*'
zstyle ':completion:*' menu select=long
zstyle ':completion:*' select-prompt %SScrolling active: current selection at %p%s
zstyle ':completion:*' use-compctl false
zstyle ':completion:*' verbose true

zstyle ':completion:*:*:kill:*:processes' list-colors '=(#b) #([0-9]#)*=0=01;31'
zstyle ':completion:*:kill:*' command 'ps -u $USER -o pid,%cpu,tty,cputime,cmd'
```

Análisis por piezas:

| Línea | Mecánica |
|---|---|
| `autoload -Uz compinit` | Carga la función del sistema `compinit` de forma **U**n sandbox (sin alias) y **z**sh-native — no la ejecuta todavía |
| `compinit` | Inicializa el sistema: indexa las funciones `_func` de completion de todos los plugins y valida su dump (`~/.zcompdump`) |
| `completer _expand _complete _correct _approximate` | **Cadena de intentos**: expandir globs/variables → completar → corregir ortografía → aproximar. Cada paso se intenta si el anterior falla |
| `menu select=2` / `select=long` | Menú interactivo: con ≥2 candidatos abre el selector (zle) — `select=long` lista solo si no cabe en una ventana |
| `matcher-list` con 4 entradas | **Múltiples estrategias en orden**: exacta → mayúsculas⇄minúsculas → al revés → separadores `._-` y cualquier parte. Pasa a la siguiente estrategia si la anterior no encuentra nada |
| `list-colors ${(s.:.)LS_COLORS}` | Divide `LS_COLORS` por `:` (`(s.:.)`) y lo adapta al formato zsh — colores de `ls` también en el completado |
| `dircolors -b` | Genera la def de `LS_COLORS` de GNU coreutils (la config personal además fija su propia `LS_COLORS`, ver [[Configuración Zsh Batcat Lsd y Burp Suite]]) |
| `use-compctl false` | Fuerza el sistema moderno `compinit` sobre el legado `compctl` |
| `kill:*` patterns | Completado de `kill`: lista procesos con `ps` coloreando los PID en rojo — seleccionas y mata sin recordar PIDs |

> [!tip] Rendimiento de `compinit`
> `compinit` tarda ~50–200 ms indexando. Optimizaciones típicas: `compinit -d ~/.cache/.zcompdump` (dump en caché), y el guard de seguridad solo-comprobar-una-vez: `compinit -u` evita el chequeo de directorios inseguros (rápido pero menos seguro), patrón popular:
> ```bash
> # comprobar seguridad solo una vez al día
> if [[ -n ${ZDOTDIR:-$HOME}/.zcompdump(#qN.mh+24) ]]; then
>   compinit
> else
>   compinit -C
> fi
> ```

## Aliases y funciones

La capa de aliases moderna (`lsd`, `batcat`) vive en [[Configuración Zsh Batcat Lsd y Burp Suite]]; las funciones `settarget`/`cleartarget` del flujo de hacking en [[Configuración de Polybar]]. Patrón recomendado para crecer sin caos: archivo dedicado `~/.zsh_aliases` sourceado desde `.zshrc`:

```bash
# al final de ~/.zshrc
[ -f ~/.zsh_aliases ] && source ~/.zsh_aliases
```

Diferencia clave alias vs función: un **alias** se expande en parse-time (el `ll` de `lsd` funciona en pipes); una **función** se resuelve en run-time y admite lógica (`if`, `$1`), como `settarget`.

## Errores comunes

1. **El plugin "no hace nada"**: falta el `source` o el guard `[ -f ]` falla (paquete no instalado). Verificar con `echo $ZSH_AUTOSUGGESTIONS_VERSION` tras sourcear.
2. **Resaltado rojo en todo**: el binario no está en el `PATH` de la sesión o el plugin se cargó antes que otros que modifican el comando.
3. **Completado viejo tras instalar un plugin**: borrar `~/.zcompdump` y reabrir (compinit cachea el índice de funciones de *todos* los plugins en el dump).
4. **`compinit` pregunta "insecure directories"**: un directorio de completado es escribible por otros usuarios; corregir con `chmod -R go-w <dir>` (¡no con `-u` a la ligera!).
5. **Instant prompt con output fantasma**: un plugin imprime al inicio; mover la impresión o desactivar instant prompt en `.p10k.zsh`.
6. **Root no hereda la config**: el `ln -s -f` se ejecutó en el CWD equivocado — verificar con `ls -l /root/.zshrc`.

## Cheatsheet

| Comando | Acción |
|---|---|
| `zsh -i -c 'echo hola'` | Simula sesión interactiva (lee `.zshrc`) |
| `echo $options` / `setopt` | Ver/cambiar opciones en vivo |
| `type cmd` / `which cmd` | Qué es `cmd` (alias/builtin/binario) |
| `bindkey` | Mapa de teclas zle (atajos de plugins) |
| `Ctrl+R` | Buscar en historial |
| `fc -l -20` | Últimos 20 comandos |
| `autoload -Uz compinit && compinit` | Reinicializar completado |
| `p10k configure` | Regenerar prompt |
| `exec zsh` | Recargar el shell completo |

## Recursos

- Manual de archivos de inicio (zsh 5.9): <https://www.zsh.org/docu/releases/zsh-5.9/doc/zshhtml/Startup_002fShutdown-Files.html>
- zsh-autosuggestions: <https://github.com/zsh-users/zsh-autosuggestions>
- zsh-syntax-highlighting: <https://github.com/zsh-users/zsh-syntax-highlighting>
- powerlevel10k: <https://github.com/romkatv/powerlevel10k>
- Guía de `zstyle`/completion: <https://zsh.sourceforge.io/Doc/Release/Zsh-Modules.html#The-completion-System>

## Relacionadas

- [[Configuración Zsh Batcat Lsd y Burp Suite]] — aliases modernos y fixes Java/Burp en el mismo `.zshrc`
- [[Configuración Post-Instalación de Entorno de Escritorio]] — dónde entra zsh en el checklist
- [[Configuración de Polybar]] — funciones zsh del flujo de targets
- [[Configuración de BSPWM y SXHKD]] — el entorno donde corre el shell
- [[Bash Scripting - Fundamentos Básicos]] — sintaxis compartida con los scripts que este shell ejecuta
- [[Comandos Básicos de Sistema y Red en Linux]] — comandos que alimentan el historial y el completado
