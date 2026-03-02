---
aliases: ["Configuración Zsh Batcat Lsd Burp Suite Fix"]
tags:
  - linux/bash
  - linux/customizacion
  - hacking/herramientas
  - herramientas/burpsuite
  - scripting/bash
estado: 🟢 Terminado
---
# Configuración de Zsh: Batcat, Lsd y Soluciones para Burp Suite

Una vez configurado, procederemos con la instalación de `batcat` y `lsd` descargando e instalando sus binarios.

## Instalación de Batcat y Lsd

> [!info] Batcat (o `bat`)
> `bat` es un clon de `cat` con superpoderes. Ofrece resaltado de sintaxis para una multitud de lenguajes de programación y archivos de configuración, integración con `git` para mostrar cambios, numeración de líneas y paginación automática a través de `less`. Es una herramienta esencial para la visualización de archivos en la terminal.

> [!info] Lsd (o `lsd`)
> `lsd` es una alternativa moderna al comando `ls`. Proporciona una salida más colorida y legible, iconos para diferentes tipos de archivos, una vista de árbol opcional y una mejor organización de la información, haciendo que la navegación por el sistema de archivos sea más intuitiva.

## Personalización de Zsh

Continuaremos personalizando Zsh con las siguientes configuraciones:

```bash
export LS_COLORS="rs=0:di=34:ln=36:mh=00:pi=40;33:so=35:do=35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=00:tw=30;42:ow=34;42:st=37;44:ex=32:*.tar=31:*.tgz=31:*.arc=31:*.arj=31:*.taz=31:*.lha=31:*.lz4=31:*.lzh=31:*.lzma=31:*.tlz=31:*.txz=31:*.tzo=31:*.t7z=31:*.zip=31:*.z=31:*.dz=31:*.gz=31:*.lrz=31:*.lz=31:*.lzo=31:*.xz=31:*.zst=31:*.tzst=31:*.bz2=31:*.bz=31:*.tbz=31:*.tbz2=31:*.tz=31:*.deb=31:*.rpm=31:*.jar=31:*.war=31:*.ear=31:*.sar=31:*.rar=31:*.alz=31:*.ace=31:*.zoo=31:*.cpio=31:*.7z=31:*.rz=31:*.cab=31:*.wim=31:*.swm=31:*.dwm=31:*.esd=31:*.avif=35:*.jpg=35:*.jpeg=35:*.mjpg=35:*.mjpeg=35:*.gif=35:*.bmp=35:*.pbm=35:*.pgm=35:*.ppm=35:*.tga=35:*.xbm=35:*.xpm=35:*.tif=35:*.tiff=35:*.png=35:*.svg=35:*.svgz=35:*.mng=35:*.pcx=35:*.mov=35:*.mpg=35:*.mpeg=35:*.m2v=35:*.mkv=35:*.webm=35:*.webp=35:*.ogm=35:*.mp4=35:*.m4v=35:*.mp4v=35:*.vob=35:*.qt=35:*.nuv=35:*.wmv=35:*.asf=35:*.rm=35:*.rmvb=35:*.flc=35:*.avi=35:*.fli=35:*.flv=35:*.gl=35:*.dl=35:*.xcf=35:*.xwd=35:*.yuv=35:*.cgm=35:*.emf=35:*.ogv=35:*.ogx=35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.m4a=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.oga=00;36:*.opus=00;36:*.spx=00;36:*.xspf=00;36:*~=00;90:*#=00;90:*.bak=00;90:*.old=00;90:*.orig=00;90:*.part=00;90:*.rej=00;90:*.swp=00;90:*.tmp=00;90:*.dpkg-dist=00;90:*.dpkg-old=00;90:*.ucf-dist=00;90:*.ucf-new=00;90:*.ucf-old=00;90:*.rpmnew=00;90:*.rpmorig=00;90:*.rpmsave=00;90:"
export PATH=/opt/kitty/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games

> [!info] `LS_COLORS`
> La variable de entorno `LS_COLORS` define los colores utilizados por el comando `ls` (y sus alternativas como `lsd`) para mostrar diferentes tipos de archivos y directorios. Cada entrada en la cadena `LS_COLORS` especifica un patrón de archivo o tipo de objeto y los códigos de color ANSI correspondientes. Esto mejora la legibilidad y la identificación visual de los elementos en el sistema de archivos.

### Alias Personalizados

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

## Solución de Problemas para Burp Suite

Burp Suite puede presentar un problema de lanzamiento; para solucionarlo, seguiremos las instrucciones detalladas en el siguiente enlace:

[Ver Solución para Burp Suite](https://medium.com/@neat_mahogany_porcupine_191/burpsuite-no-longer-launches-after-parrot-upgrade-d1c6b17cb70d)

Una vez resuelto el problema de lanzamiento de Burp Suite, es posible que la interfaz se muestre desproporcionada. Para corregir esto, añadiremos la siguiente línea a nuestro archivo `.zshrc`:

```bash
# Fix the Java Problem
export _JAVA_AWT_WM_NONREPARENTING=1
```

> [!tip] `_JAVA_AWT_WM_NONREPARENTING=1`
> Esta variable de entorno es crucial para las aplicaciones Java que utilizan AWT (Abstract Window Toolkit), especialmente en entornos de ventanas tipo *tiling window managers* (como `i3`, `bspwm`, `AwesomeWM`). Al establecerla en `1`, se indica a Java que no intente "reparentar" las ventanas, lo que a menudo causa problemas de visualización, como ventanas desproporcionadas o que no se muestran correctamente, al interactuar con gestores de ventanas que no siguen el modelo tradicional de reparenting.

Adicionalmente, para una correcta gestión de ventanas de aplicaciones Java en entornos como `bspwm`, agregaremos la siguiente línea a la configuración de `bspwm`:

```bash
wmname LG3D &
```

> [!info] `wmname LG3D`
> El comando `wmname` se utiliza para establecer el nombre del gestor de ventanas que se reporta a las aplicaciones. Algunas aplicaciones Java, como Burp Suite, tienen problemas para detectar correctamente el gestor de ventanas en entornos *tiling* o no estándar. Al establecer `wmname LG3D`, se simula que el gestor de ventanas es "LG3D", un gestor de ventanas de Java 3D. Esto a menudo engaña a las aplicaciones Java para que se comporten de manera más predecible y muestren sus ventanas correctamente, evitando problemas de tamaño o renderizado. El `&` al final permite que el comando se ejecute en segundo plano.

Para que los cambios surtan efecto, es necesario cerrar la sesión actual y volver a iniciarla.