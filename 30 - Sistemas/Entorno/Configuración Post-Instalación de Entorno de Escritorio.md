---
tags:
  - sistemas
  - entorno
  - post-instalacion-escritorio
tipo: concepto
estado: terminado
---

# Configuración Post-Instalación de Entorno de Escritorio

Checklist reproducible de lo que se configura **después** de tener bspwm corriendo (arranque y WM en [[Configuración de BSPWM y SXHKD]]): aplicaciones, tipografía, shell, terminal, portapapeles y fondo. Cada paso incluye los comandos exactos y el porqué — el objetivo es poder reconstruir la máquina completa siguiendo esta nota de arriba a abajo.

> [!info] Filosofía del checklist
> Un entorno de mosaico es un **ensamblaje**: el WM solo coloca ventanas; todo lo demás (terminal, shell, fuentes, fondo) se instala y configura por separado y se conecta vía `bspwmrc` y `sxhkdrc`. El orden aquí importa: las fuentes antes que polybar/kitty (los glifos dependen de ellas), el shell antes que la config de kitty (`shell zsh`).

Orden de ejecución recomendado:

| # | Paso | Depende de |
|---|---|---|
| 1 | Atajo de Firefox en `sxhkdrc` | sxhkd corriendo |
| 2 | Hack Nerd Fonts | — |
| 3 | Zsh + plugins | — |
| 4 | Portapapeles dual (VMware) | VMware Tools |
| 5–6 | Kitty + config | Paso 2 (glifos), paso 3 (`shell zsh`) |
| 7 | Fondo con `feh` | — |

## 1. Lanzador web en `sxhkdrc`

Definir atajos para lanzar Firefox u otras apps web desde sxhkd, el demonio de atajos de [[Configuración de BSPWM y SXHKD|sxhkd]]:

```bash
# browser
super + {_,shift + }w
	{firefox,firefox --private-window}
```

> [!info] `sxhkd` (Simple X Hotkey Daemon)
> Herramienta ligera que traduce chords de teclado en comandos (`sh -c`), sin estar atada a ningún WM. Fundamental en entornos *tiling* para lanzar aplicaciones, cambiar de escritorio o controlar el sistema sin ratón. Config: `~/.config/sxhkd/sxhkdrc`; recarga en caliente con `pkill -USR1 -x sxhkd`. La sintaxis completa (`{a,b}` expansiones, chords) está en [[Configuración de BSPWM y SXHKD]].

## 2. Instalar las Hack Nerd Fonts

Las **Nerd Fonts** son fuentes parcheadas que inyectan +10 000 glifos de iconos: Font Awesome, Devicons, Octicons, Material Icons, y los glifos "pomodoro" que usa [[Configuración de Polybar|polybar]] (` 󰌽 󰂉`), la prompt de p10k y los iconos de `lsd`. Sin ellas, todos esos iconos se dibujan como cajas □.

Instalación manual (la forma clásica):

```bash
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts
# descargar Hack.zip desde https://www.nerdfonts.com/font-downloads y descomprimir:
unzip Hack.zip
fc-cache -fv            # reconstruye la caché de fontconfig
fc-list | grep -i hack  # verifica: debe listar "Hack Nerd Font"
```

| Paso | Por qué |
|---|---|
| `~/.local/share/fonts/` | Directorio XDG de fuentes **de usuario** — no necesita root y sobrevive a cambios de sistema |
| `fc-cache -fv` | fontconfig mantiene una **caché** de fuentes; sin reconstruirla, las apps no ven las nuevas (`-v` muestra qué directorios escanea) |
| `fc-list` | Lista las fuentes realmente registradas — la primera línea de defensa cuando "no se ve el icono" |

> [!tip] Alternativa por paquete
> Debian/Ubuntu empacan `fonts-hack-ttf` (`sudo apt install fonts-hack-ttf`), pero **sin** los glifos Nerd (es la fuente base). Para el entorno de estas notas se necesita la variante *Hack Nerd Font* de la web oficial. Otras fuentes del setup personal: Iosevka Nerd Font, Source Code Pro, Montserrat (ver `font-*` de [[Configuración de Polybar]]).

## 3. Instalar Zsh

Zsh es el shell interactivo del entorno: autocompletado avanzado, corrección ortográfica, historial compartido y ecosistema de plugins.

```bash
sudo apt install zsh zsh-autosuggestions zsh-syntax-highlighting zsh-sudo
chsh -s $(which zsh)     # zsh como shell por defecto del usuario
```

> [!info] Ventajas sobre bash
> Globbing recursivo (`**`), sustitución de expansión, completado con menús seleccionables (zle), `setopt` de comportamiento fino y mejoras de rendimiento en líneas largas. La configuración completa — Powerlevel10k, plugins, historial, completado `zstyle`, enlace simbólico para root — vive en [[Configuración de Zsh y Plugins]].

## 4. Activar el portapapeles dual (VMware)

Para copiar y pegar entre host y máquina virtual hay que arrancar el agente de integración de VMware Tools en cada sesión.

### 4.1. Añadir en `.config/bspwm/bspwmrc`:

```bash
vmware-user-suid-wrapper &
```

> [!warning] `vmware-user-suid-wrapper`
> Binario de VMware Tools (suid) que lanza `vmware-user` — el agente que implementa el **vmmouse/vmware** X11 drivers: portapapeles compartido, arrastrar-y-soltar, ajuste automático de resolución y teclado liberado con Ctrl+Alt. El `&` es obligatorio: es un proceso persistente y bloquearía el resto de `bspwmrc` (ver [[Configuración de BSPWM y SXHKD]]).
>
> Diagnóstico: si el portapapeles no cruza, verificar `ps aux | grep vmware-user` y que el paquete `open-vm-tools-desktop` esté instalado (Debian: `sudo apt install open-vm-tools-desktop` — en algunas instalaciones basta con este paquete y no el wrapper antiguo).

## 5. Instalar Kitty (versión actualizada)

```bash
sudo apt install kitty
```

**Kitty** es un emulador de terminal acelerado por GPU (OpenGL): renderiza texto e iconos en la tarjeta, soporta ligaduras, pestañas, splits, buffers de portapapeles múltiples y transparencia — todo configurable desde texto plano.

> [!info] Características
> Su renderizado por GPU elimina el lag de scroll en terminales pesadas (logs, htop), y su protocolo de terminado (kitty keyboard protocol) permite atajos como los `map ctrl+f1 copy_to_buffer` de la config personal. Config: `~/.config/kitty/kitty.conf` + archivos `include`ados.

## 6. Configuración de Kitty

Dos archivos: `color.ini` (paleta) y `kitty.conf` (ajustes), en `~/.config/kitty/`.

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

> [!tip] Paleta Tokyo Night
> Los hex (`#1a1b26`, `#7aa2f7`, `#73daca`…) son la paleta **Tokyo Night**: `color0–15` mapean los 16 colores ANSI que las apps de terminal usan con `tput setaf N`/`LS_COLORS`. `cursor_shape Underline` + grosor 1 dan el cursor subrayado fino; `window_padding_width 20` añade aire alrededor del texto. Estos colores alimentan también el completado con `LS_COLORS` (ver [[Configuración Zsh Batcat Lsd y Burp Suite]]).

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

> [!info] Detalles de la config
> - `enable_audio_bell no`: silencia la campana (X11 bell).
> - `include color.ini`: importa la paleta — kitty includea cualquier otro archivo de config, lo que permite mantener temas separados.
> - `font_family HackNerdFont`, `font_size 13`: la Nerd Font del paso 2 es obligatoria para los glifos de polybar/p10k dentro de la terminal.
> - `disable_ligatures never`: `->`, `=>` y consortes se dibujan como ligaduras.
> - `url_color` / `url_style curly`: URLs detectadas (`detect_urls yes`) en azul y con llaves rizadas.
> - `map ctrl+flecha neighboring_window ...`: navegación entre splits como en tmux; `map ctrl+shift+z toggle_layout stack` alterna split ↔ stack.
> - `copy_to_buffer a/b` en F1–F4: **dos portapapeles internos** independientes del del sistema.
> - `repaint_delay 10` / `input_delay 3` / `sync_to_monitor yes`: tuning de render (ms entre repintados; latencia de input; sincronía con el refresco).
> - `tab_bar_style powerline` + colores de pestaña (`#98c379` activa): barra estilo powerline.
> - `new_window_with_cwd` (Ctrl+Shift+Enter): abre splits/pestañas **en el directorio actual** — evita el `cd` repetido.
> - `background_opacity 0.95`: transparencia sutil — real gracias a [[Configuración de Picom|picom]].
> - `shell zsh`: las ventanas de kitty arrancan zsh directamente (no pasan por el shell de login por defecto).

## 7. Fondo de pantalla con `feh`

`feh` es un visor de imágenes ligero que sirve además de fijador de fondos para WMs sin escritorio propio.

```bash
feh --bg-fill ~/wallpaper.jpg
```

| Opción | Efecto |
|---|---|
| `--bg-fill` | Escala para llenar sin distorsión, **recortando** los sobrantes (la más usada) |
| `--bg-max` | Escala para caber, sin recortar (puede dejar bandas) |
| `--bg-tile` | Mosaico |
| `--no-fehbg` | No escribe `~/.fehbg` (el script de restauración que feh guarda) |

Para que cargue con la sesión, va en `~/.config/bspwm/bspwmrc`:

```bash
feh --bg-fill ~/wallpaper.jpg &
```

> [!tip] `~/.fehbg`
> Cada vez que fijas un fondo, feh guarda el comando exacto en `~/.fehbg` (un script shell). Alternativa a re-ejecutar feh: añadir `~/.fehbg &` al `bspwmrc` — restaura el último fondo sin re-escalar la imagen cada arranque.

## Lista de comandos (resumen ejecutable)

```bash
# 2. fuentes
mkdir -p ~/.local/share/fonts && cd ~/.local/share/fonts && unzip Hack.zip && fc-cache -fv
# 3. shell
sudo apt install zsh zsh-autosuggestions zsh-syntax-highlighting zsh-sudo && chsh -s $(which zsh)
# 4. portapapeles VM (además en bspwmrc)
sudo apt install open-vm-tools-desktop
# 5. terminal
sudo apt install kitty
# 7. fondo (además en bspwmrc)
feh --bg-fill ~/wallpaper.jpg
```

## Errores comunes

1. **Cajas □ donde debería haber iconos**: fuentes Nerd no instaladas o `fc-cache` no corrido; verificar con `fc-list | grep -i nerd`.
2. **kitty dice "Unable to set font"**: el `font_family` no coincide con el nombre registrado (`fc-list | grep Hack` → "Hack Nerd Font" puede requerir el nombre completo).
3. **`chsh` no cambia el shell**: en Debian el binario debe estar en `/etc/shells` (`grep zsh /etc/shells`).
4. **Portapapeles VMware muere tras un rato**: `vmware-user-suid-wrapper` sin `&` en `bspwmrc` o duplicado — debe lanzarse una sola vez por sesión.
5. **Transparencia negra en kitty**: falta el compositor ([[Configuración de Picom]]) — sin él kitty dibuja opaco (o grañoso con pseudo-transparencia).
6. **Fondo vuelve al negro tras `bspc wm -r`**: falta `feh` en `bspwmrc`; el WM no restaura fondos solo.

## Relacionadas

- [[Configuración de BSPWM y SXHKD]] — WM, sxhkd y `bspwmrc` (donde se conectan pasos 1, 4 y 7)
- [[Configuración de Zsh y Plugins]] — el paso 3 en profundidad
- [[Configuración Zsh Batcat Lsd y Burp Suite]] — `batcat`/`lsd`/`fzf` y fixes Java/Burp
- [[Configuración de Polybar]] — consumidor directo de las Nerd Fonts
- [[Instalación de Polybar y Picom]] — el resto de la pila gráfica
- [[Vim Comandos Esenciales]] — el editor con el que se editan todos estos configs
- [[Comandos Básicos de Sistema y Red en Linux]] — comandos base usados en el checklist
