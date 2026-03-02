---
aliases:
  - Configuración de Entorno de Escritorio
  - Post-instalación Bspwm
tags:
  - linux/customizacion
estado: 🟢 Terminado
---
# Configuración Post-Instalación de Entorno de Escritorio

Continuando con la configuración iniciada en, a continuación se enumeran los pasos a seguir para agilizar el proceso de personalización del entorno de escritorio:

## 1. Configurar Firefox en `sxhkdrc`

Este paso implica definir atajos de teclado para lanzar Firefox u otras aplicaciones web utilizando `sxhkd`, el demonio de atajos de teclado simple de X.

> [!info] `sxhkd` (Simple X Hotkey Daemon)
> `sxhkd` es una herramienta ligera que permite definir atajos de teclado y comandos asociados. Es fundamental en entornos de gestores de ventanas tipo *tiling* como `bspwm` para lanzar aplicaciones, cambiar de escritorio, o controlar el sistema sin usar el ratón. La configuración se realiza en el archivo `~/.config/sxhkd/sxhkdrc`.

## 2. Instalar las Hack Nerd Fonts

Las Nerd Fonts son una colección de fuentes parcheadas que incluyen una gran cantidad de glifos adicionales de conjuntos populares como Font Awesome, Devicons, Octicons, y más. Son esenciales para una visualización correcta de iconos en terminales, barras de estado (como Polybar) y editores de texto.

> [!tip] Instalación de Nerd Fonts
> Para instalar las Hack Nerd Fonts, generalmente se descargan los archivos `.ttf` o `.otf` y se copian a `~/.local/share/fonts/` o `/usr/share/fonts/`. Después, es necesario reconstruir la caché de fuentes con `fc-cache -fv`.

## 3. Instalar Zsh

Zsh (Z Shell) es un potente shell de línea de comandos que ofrece numerosas mejoras sobre Bash, incluyendo autocompletado avanzado, corrección ortográfica, temas y un robusto sistema de plugins (como Oh My Zsh).

> [!info] Ventajas de Zsh
> Zsh es conocido por su extensibilidad y características avanzadas que mejoran la productividad. Permite una personalización profunda a través de frameworks como Oh My Zsh, que facilita la gestión de plugins y temas, ofreciendo una experiencia de terminal más rica y eficiente.

## 4. Activar el portapapeles dual

Para asegurar la funcionalidad de copiar y pegar entre el sistema anfitrión y una máquina virtual (especialmente en entornos VMware), es necesario activar el soporte del portapapeles dual.

### 4.1. Añadir el siguiente comando en `.config/bspwm/bspwmrc`:

```bash
vmware-user-suid-wrapper &
```

> [!warning] `vmware-user-suid-wrapper`
> Este comando es parte de las VMware Tools y es crucial para habilitar la integración de características entre el sistema operativo invitado y el anfitrión, incluyendo el portapapeles compartido, el arrastrar y soltar, y el ajuste automático de la resolución de pantalla. El `&` al final permite que el comando se ejecute en segundo plano, sin bloquear el inicio del gestor de ventanas.

## 5. Instalar Kitty (versión actualizada)

Kitty es un emulador de terminal rápido y rico en características, acelerado por GPU. Es una excelente alternativa a otros emuladores de terminal por su rendimiento y capacidad de personalización.

> [!info] Características de Kitty
> Kitty se destaca por su renderizado acelerado por GPU, lo que resulta en una experiencia de terminal muy fluida, especialmente con fuentes y animaciones. Soporta *ligatures*, pestañas, ventanas divididas y una configuración muy flexible a través de su archivo `kitty.conf`.

## 6. Configuración de Kitty

La configuración de Kitty se realiza principalmente a través de dos archivos: `color.ini` para la paleta de colores y `kitty.conf` para el resto de los ajustes.

### 6.1. Archivo `color.ini`:

```bash
cursor_shape          Underline
cursor_underline_thickness 1
window_padding_width  20

# Special
foreground #a9b1d6
background #1a1b26

# Black
color0 #414868
color8 #414868

# Red
color1 #f7768e
color9 #f7768e

# Green
color2  #73daca
color10 #73daca

# Yellow
color3  #e0af68
color11 #e0af68

# Blue
color4  #7aa2f7
color12 #7aa2f7

# Magenta
color5  #bb9af7
color13 #bb9af7

# Cyan
color6  #7dcfff
color14 #7dcfff

# White
color7  #c0caf5
color15 #c0caf5

# Cursor
cursor #c0caf5
cursor_text_color #1a1b26

# Selection highlight
selection_foreground #7aa2f7
selection_background #28344a
```

> [!tip] Paleta de Colores
> Este bloque define una paleta de colores específica, a menudo inspirada en temas populares como "Tokyo Night" o "Dracula". `foreground` y `background` establecen los colores predeterminados del texto y el fondo. Los `color0` a `color15` definen los colores para los 16 colores ANSI estándar, utilizados por muchas aplicaciones de terminal. `cursor_shape` y `cursor_underline_thickness` personalizan la apariencia del cursor.

### 6.2. Archivo `kitty.conf`:

```bash
enable_audio_bell no

include color.ini

font_family HackNerdFont
font_size 13

disable_ligatures never

url_color #61afef

url_style curly

map ctrl+left neighboring_window left
map ctrl+right neighboring_window right
map ctrl+up neighboring_window up
map ctrl+down neighboring_window down

map f1 copy_to_buffer a
map f2 paste_from_buffer a
map f3 copy_to_buffer b
map f4 paste_from_buffer b

cursor_shape beam
cursor_beam_thickness 1.8

mouse_hide_wait 3.0
detect_urls yes

repaint_delay 10
input_delay 3
sync_to_monitor yes

map ctrl+shift+z toggle_layout stack
tab_bar_style powerline

inactive_tab_background #e06c75
active_tab_background #98c379
inactive_tab_foreground #000000
tab_bar_margin_color black

map ctrl+shift+enter new_window_with_cwd
map ctrl+shift+t new_tab_with_cwd

background_opacity 0.95

shell zsh
```

> [!info] Detalles de Configuración de Kitty
> *   `enable_audio_bell no`: Desactiva el sonido de la campana del terminal.
> *   `include color.ini`: Importa la configuración de colores desde el archivo `color.ini`.
> *   `font_family HackNerdFont`, `font_size 13`: Especifica la fuente y su tamaño. Es crucial usar una Nerd Font para la correcta visualización de glifos.
> *   `disable_ligatures never`: Habilita las ligaduras de fuentes, que combinan caracteres como `->` en un solo glifo.
> *   `url_color`, `url_style`: Personaliza la apariencia de los enlaces detectados.
> *   `map`: Define atajos de teclado personalizados. Aquí se configuran atajos para navegar entre ventanas (`neighboring_window`) y para gestionar múltiples portapapeles internos de Kitty (`copy_to_buffer`, `paste_from_buffer`).
> *   `cursor_shape beam`, `cursor_beam_thickness`: Configura el cursor como una barra vertical delgada.
> *   `detect_urls yes`: Habilita la detección automática de URLs.
> *   `tab_bar_style powerline`: Aplica un estilo "powerline" a la barra de pestañas, que es visualmente atractivo.
> *   `inactive_tab_background`, `active_tab_background`, `inactive_tab_foreground`: Colores para las pestañas activas e inactivas.
> *   `new_window_with_cwd`, `new_tab_with_cwd`: Atajos para abrir nuevas ventanas o pestañas en el directorio de trabajo actual.
> *   `background_opacity 0.95`: Establece la transparencia del fondo del terminal.
> *   `shell zsh`: Configura Zsh como el shell predeterminado para las nuevas instancias de Kitty.

## 7. Configurar el fondo de pantalla con `feh`

`feh` es un visor de imágenes ligero y rápido que también se utiliza comúnmente para establecer fondos de pantalla en gestores de ventanas minimalistas.

> [!tip] Uso de `feh` para fondos de pantalla
> Para establecer un fondo de pantalla, se usa un comando como `feh --bg-fill /ruta/a/tu/imagen.jpg`. Para que el fondo de pantalla se cargue automáticamente al inicio del sistema, este comando se suele añadir al archivo de configuración del gestor de ventanas (ej. `~/.config/bspwm/bspwmrc`) o a un script de inicio. La opción `--bg-fill` escala la imagen para llenar la pantalla sin distorsión.