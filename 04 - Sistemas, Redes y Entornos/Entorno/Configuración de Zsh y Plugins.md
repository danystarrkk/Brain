---
aliases:
  - Zsh Config
  - Configuración Zsh
  - Zsh Plugins
tags:
  - linux/customizacion
estado: 🟢 Terminado
---
# Configuración Avanzada de Zsh y Plugins

Continuando con la configuración, vamos a comenzar ejecutando los siguientes comandos para instalar plugins esenciales de Zsh:

```bash
sudo apt install zsh-autosuggestions zsh-syntax-highlighting
```

> [!info] **Plugins Esenciales de Zsh**
> *   **`zsh-autosuggestions`**: Este plugin sugiere comandos a medida que escribes, basándose en tu historial de comandos. Las sugerencias aparecen en gris y pueden ser aceptadas presionando la flecha derecha (`→`).
> *   **`zsh-syntax-highlighting`**: Proporciona resaltado de sintaxis para los comandos en la línea de comandos. Esto ayuda a identificar errores tipográficos y a visualizar la estructura del comando antes de ejecutarlo. Por ejemplo, los comandos válidos pueden aparecer en verde, mientras que los inválidos o los argumentos pueden tener otros colores.

Luego, se debe instalar [Powerlevel10k](https://github.com/romkatv/powerlevel10k).

> [!tip] **Powerlevel10k (p10k)**
> Powerlevel10k es un tema altamente personalizable para Zsh. Ofrece un rendimiento excepcional y una gran cantidad de opciones para configurar el prompt, incluyendo información contextual como el directorio actual, el estado de Git, el tiempo de ejecución del último comando, etc. Su configuración se realiza a través del comando `p10k configure` después de la instalación.

Realizar la configuración básica de Zsh y Powerlevel10k tanto para el usuario actual como para el usuario `root`.

Por último, realizar el enlace simbólico con el siguiente comando desde la carpeta `root` para sincronizar la configuración del usuario con la de `root`:

```bash
ln -s -f /home/stark/.zshrc .zshrc
```

> [!info] **Enlace Simbólico (`ln -s -f`)**
> *   **`ln`**: Comando para crear enlaces (links).
> *   **`-s` (symbolic)**: Crea un enlace simbólico (soft link), que es un puntero a otro archivo o directorio. A diferencia de un enlace duro, un enlace simbólico puede apuntar a archivos en diferentes sistemas de archivos y funciona incluso si el archivo original se mueve (aunque el enlace se rompería si el destino cambia de ubicación).
> *   **`-f` (force)**: Si el archivo de destino ya existe, lo elimina antes de crear el nuevo enlace.
> *   En este caso, se crea un enlace simbólico llamado `.zshrc` en el directorio `root` (`/root/`) que apunta al archivo de configuración `.zshrc` del usuario `stark` (`/home/stark/.zshrc`). Esto asegura que ambos usuarios compartan la misma configuración de Zsh.

Ya con esto, agreguemos lo siguiente en el archivo `.zshrc`, recordando hacerlo después de la configuración de Powerlevel10k. También es necesario instalar el plugin `zsh-sudo`.

> [!warning] **Orden de Carga de Plugins**
> Es crucial que la configuración de los plugins de Zsh se realice en el orden correcto dentro del archivo `.zshrc`. Algunos plugins, como `zsh-autosuggestions` y `zsh-syntax-highlighting`, deben cargarse *después* de la configuración del tema (como Powerlevel10k) para asegurar que funcionen correctamente y no interfieran con el prompt.

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

# Configuración del Historial de Comandos
# Define el archivo de historial, el tamaño y las opciones de comportamiento.
HISTFILE=~/zsh_history
HISTSIZE=10000
SAVEHIST=10000 # Corregido: SAVEHIST es el que limita cuántas entradas se guardan en el archivo.
setopt histignorealldups sharehistory

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

> [!info] **Configuración del Historial de Zsh**
> *   **`HISTFILE=~/zsh_history`**: Especifica la ruta del archivo donde se guardará el historial de comandos. `~/` se expande al directorio home del usuario.
> *   **`HISTSIZE=10000`**: Define el número máximo de comandos que Zsh mantendrá en la memoria durante la sesión actual.
> *   **`SAVEHIST=10000`**: Define el número máximo de comandos que se guardarán en el archivo especificado por `HISTFILE` al finalizar la sesión. (El original tenía `HISTLIST=10000` que no es una variable estándar de Zsh para este propósito; `SAVEHIST` es la correcta).
> *   **`setopt histignorealldups`**: Evita que se guarden comandos duplicados consecutivos en el historial. Si ejecutas el mismo comando varias veces seguidas, solo se guardará una instancia.
> *   **`setopt sharehistory`**: Permite que múltiples sesiones de Zsh compartan el mismo archivo de historial, sincronizando los comandos entre ellas.

> [!tip] **Configuración del Sistema de Completado de Zsh (`zstyle`)**
> El sistema de completado de Zsh es extremadamente potente y personalizable. Los comandos `zstyle` se utilizan para configurar su comportamiento:
> *   **`autoload -Uz compinit`**: Carga la función `compinit` de forma universal y sin alias, que es responsable de inicializar el sistema de completado.
> *   **`compinit`**: Ejecuta la inicialización del sistema de completado.
> *   **`zstyle ':completion:*' ...`**: Estas líneas configuran varios aspectos del completado, como:
>     *   **`auto-description`**: Muestra descripciones de los elementos completados.
>     *   **`completer`**: Define la secuencia de funciones de completado a intentar (`_expand` para expansión de alias, `_complete` para completado normal, `_correct` para corrección de errores, `_approximate` para completado aproximado).
>     *   **`format` / `group-name`**: Controlan el formato de la salida del completado.
>     *   **`menu select`**: Determina cómo se seleccionan los elementos del menú de completado.
>     *   **`eval "$(dircolors -b)"`**: Carga las configuraciones de color para `ls` en el completado.
>     *   **`matcher-list`**: Define reglas de coincidencia para el completado, permitiendo, por ejemplo, la coincidencia insensible a mayúsculas y minúsculas o la coincidencia de guiones bajos/guiones.
>     *   **`list-prompt` / `select-prompt`**: Personalizan los mensajes que aparecen durante el completado interactivo.
>     *   **`use-compctl false`**: Deshabilita el sistema de completado antiguo (`compctl`) en favor del moderno.
>     *   **`verbose true`**: Muestra información adicional durante el completado.
> *   **`zstyle ':completion:*:kill:*' command 'ps -u $USER -o pid,%cpu,tty,cputime,cmd'`**: Un ejemplo específico de `zstyle` que personaliza el completado para el comando `kill`. En lugar de solo mostrar PIDs, muestra una lista detallada de procesos del usuario, lo que facilita la selección del proceso correcto para terminar.