---
aliases:
  - Blue Sky Polybar
  - Tema blue-sky
  - Vaughn Valle polybar
tags:
  - sys/entorno
  - linux/customizacion
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Configuración de Polybar]]"
  - "[[Instalación de Polybar y Picom]]"
  - "[[Configuración de BSPWM y SXHKD]]"
  - "[[Configuración Post-Instalación de Entorno de Escritorio]]"
fuente: "https://github.com/VaughnValle/blue-sky"
---

# Instalación y Configuración de Blue Sky Polybar

**blue-sky** es el tema de [[Configuración de Polybar|Polybar]] de [Vaughn Valle](https://github.com/VaughnValle/blue-sky): un set de barras flotantes con esquinas redondeadas, fondo translúcido, tipografía Iosevka y acento azul — pensado para acompañarse con picom (esquinas + transparencia) y wallpapers de tono cielo. Es el **punto de partida** del setup personal: la config actual de `current.ini`/`workspace.ini` ([[Configuración de Polybar]]) evolucionó de este tema — sus barras flotantes, el patrón `inherit`, los separadores y la paleta son herencia directa de blue-sky.

> [!info] Qué contiene el repo
> `blue-sky/polybar/` trae el archivo de config de polybar, `colors.ini` (paleta), `launch.sh` (lanzador que mata y relanza las barras) y `fonts/` (las Iosevka Nerd Font que el tema usa). El repo original incluye además su wallpaper y su sección de picom — la instalación siguiente se centra en la parte de polybar.

## Instalación

Ejecutar en el directorio de descargas:

```bash
git clone https://github.com/VaughnValle/blue-sky.git
cd blue-sky/polybar/ 
cp -r * ~/.config/polybar
echo '~/.config/polybar/./launch.sh &' >> ~/.config/bspwm/bspwmrc
sudo cp fonts/* /usr/share/fonts/truetype
fc-cache -v
```

> [!info] Polybar
> Polybar es una barra de estado ligera y altamente personalizable diseñada para sistemas Linux. Se utiliza comúnmente con gestores de ventanas de mosaico (tiling window managers) como `bspwm`, `i3`, `AwesomeWM`, etc., para mostrar información del sistema (CPU, RAM, red), notificaciones, controles de volumen, y más, proporcionando una interfaz de usuario limpia y funcional.

## Explicación detallada de los comandos

### 1. `git clone https://github.com/VaughnValle/blue-sky.git`

> [!info] **`git clone`**
> Descarga una copia completa del repositorio Git remoto a la máquina local, incluyendo historial de versiones. Crea el directorio `blue-sky/` con el estado actual del tema.

### 2. `cd blue-sky/polybar/`

> [!info] **`cd`**
> Cambia al subdirectorio `polybar/` del repo, donde están los archivos de configuración específicos de la barra (los `.ini`, el lanzador y las fuentes).

### 3. `cp -r * ~/.config/polybar`

> [!info] **`cp -r`**
> Copia **recursivamente** (`-r`) todos los archivos y directorios del tema a `~/.config/polybar/`, la ubicación XDG estándar de la config de polybar.
>
> [!warning] Sobrescritura
> `cp` no pregunta: cualquier `config.ini`, `colors.ini` o `launch.sh` **previo** en `~/.config/polybar/` queda reemplazado. Antes de aplicar un tema sobre un entorno ya configurado, respaldar el directorio (`cp -r ~/.config/polybar ~/.config/polybar.bak`). En el setup personal este paso se hizo una vez como punto de partida y la config se ha mantenido/editado desde entonces.

### 4. `echo '~/.config/polybar/./launch.sh &' >> ~/.config/bspwm/bspwmrc`

> [!warning] **Persistencia en `bspwmrc`**
> Añade al final de `~/.config/bspwm/bspwmrc` (el script de arranque de [[Configuración de BSPWM y SXHKD|bspwm]]) la línea que lanza las barras del tema:
> - `>>` — append: añade sin sobrescribir el resto del script.
> - `~/.config/polybar/./launch.sh` — ruta al lanzador del tema (el `./` intermedio es redundante pero inofensivo; es la forma en que el tema original lo documenta).
> - `&` — lanza en segundo plano: sin él, `bspwmrc` esperaría a polybar (proceso persistente) y el WM quedaría bloqueado a medio arrancar.
>
> Ojo con las **comillas simples**: el `~` debe expandirse en tiempo de ejecución del `bspwmrc`, no del echo — con `~` escapado por comillas en el archivo final funciona porque `sh` lo expande al ejecutar la línea; igualmente válido y más claro es escribir directamente la ruta absoluta.

### 5. `sudo cp fonts/* /usr/share/fonts/truetype`

> [!tip] **Instalación de fuentes del sistema**
> Copia las fuentes TrueType del tema a `/usr/share/fonts/truetype/`, disponible para **todos los usuarios y apps** del sistema. `sudo` porque es un directorio del sistema. Alternativa sin root (ver [[Configuración Post-Instalación de Entorno de Escritorio]]): `~/.local/share/fonts/`.

### 6. `fc-cache -v`

> [!info] **Reconstrucción de la caché de fuentes**
> fontconfig mantiene un índice de fuentes; tras instalar nuevas hay que reconstruirlo con `fc-cache` (`-v` = verbose, muestra cada directorio escaneado). Sin este paso, polybar sigue "sin encontrar" la fuente del tema. Verificación: `fc-list | grep -i iosevka`.

## Anatomía del tema: cómo funciona por dentro

| Archivo | Rol | Relación con la config personal |
|---|---|---|
| `config.ini` | Barras del tema (`[bar/main]` + barras flotantes) | Antecesor directo del patrón `inherit = bar/main` de `current.ini` |
| `colors.ini` | Sección `[color]` con la paleta (azules cielo + fondos translúcidos ARGB) | Seguía siendo el `include-file` de la config personal |
| `launch.sh` | `killall -q polybar` → espera a que mueran → lanza cada barra con `&` | El `launch.sh` personal (documentado en [[Configuración de Polybar]]) hereda exactamente ese esqueleto |
| `fonts/` | Iosevka Nerd Font del tema | La familia `font-0 = "Iosevka Nerd Font…"` de la config actual |

Puntos de diseño del tema que vale la pena extraer:

- **Barras flotantes** (`offset-y` > 0, `override-redirect` o `wm-restack`): no pegan al borde — flotan con margen, combinando con las esquinas redondeadas de picom.
- **Transparencia por compositor**: el fondo ARGB del `colors.ini` solo se ve correctamente con [[Configuración de Picom|picom]] corriendo; sin él, pseudo-transparencia o opaco.
- **Todo en un `.ini` + colores separados**: cambiar de tema = cambiar `colors.ini`, no tocar geometrías.

### El visual de blue-sky

El tema se reconoce a simple vista: segmentos de barra **separados** (izquierda para workspaces, centro para reloj/título, derecha para sistema) flotando sobre el wallpaper con margen uniforme, esquinas redondeadas (el `corner-radius` de picom hace el resto), fondo oscuro translúcido y texto/acentos en azul claro estilo "cielo" — de ahí el nombre. Los separadores entre módulos son espacios anchos, no bordes: cada bloque parece una isla. Ese look es exactamente el que heredó la disposición multi-barras del setup personal (fechas, ethernet, VPN y target cada uno en su propia "isla" en el borde superior).

### Receta: cambiar el color de acento

El color del tema vive en `~/.config/polybar/colors.ini` (sección `[color]`), referenciado por `${color.ac}` / `${color.blue}` en los `.ini`:

```ini
[color]
; estructura típica — los valores exactos son los de tu colors.ini actual
ac = #61a9ef        ; color de acento (hex)
bg = #ee1a1b26      ; fondo translúcido #AARRGGBB — AA=ee ≈ 93 % de opacidad
```

1. Elegir el hex nuevo (con canal alfa si se quiere translúcido: `#AARRGGBB`).
2. Reemplazar el valor en `colors.ini` — todos los `${color.ac}` de las barras lo heredan.
3. Relanzar: `killall -q polybar; ~/.config/polybar/launch.sh` (o `bspc wm -r`).

> [!tip] Un solo archivo, todo el tema
> Precisamente por el patrón `include-file = ~/.config/polybar/colors.ini`, retocar un valor de `colors.ini` cambia workspaces, fecha, VPN y menú a la vez — la lección de arquitectura que el setup personal adoptó y expandió con su propio `colors.ini`.

### Receta: añadir un módulo al tema

1. Definir el módulo en `config.ini` (o en `current.ini` personal), p. ej. un `custom/text`:

```ini
[module/web]
type = custom/text
content = " "
click-left = firefox &
```

2. Agregarlo a un bloque de barra: `modules-left = workspaces web`.
3. Probar en foreground antes de tocar el lanzador: `polybar main -c ~/.config/polybar/config.ini -l info` — el log reporta typos de sección con la línea exacta.
4. Si el icono sale en caja □ → falta la Nerd Font o el índice de fuente equivocado (`%{T1}` si el glifo está en `font-1`).

Catálogo completo de tipos de módulo, tags `%{A…}`/`%{T…}` y claves de `[bar/*]` en [[Configuración de Polybar]].

### blue-sky y picom: la pareja inseparable

El look del tema **asume un compositor**: las esquinas redondeadas de las barras se logran con el `corner-radius` de picom recortando las ventanas de la barra, y la translucidez necesita composición real (ver [[Configuración de Picom]]). El repo original de blue-sky incluye además su propio bloque de picom emparejado con el tema. En este entorno, el `picom.conf` personal (esquinas 20 px, blur dual_kawase) cumple ese rol. Sin picom: barras rectas y fondo opaco — el tema "funciona" pero pierde la mitad de la estética.

## Del blue-sky original al setup personal

El vault no usa blue-sky "tal cual": la config de [[Configuración de Polybar]] es una **evolución**. El mapa de herencia, para no perder la relación al leer ambos archivos:

| Elemento de blue-sky | Qué hizo el setup personal |
|---|---|
| `[bar/main]` base + `inherit` | Adoptado y extendido a **7 barras heredadas** (primary, ethernet_bar, target_hack, vpn_bar, secondary, log, top) |
| Barras flotantes superiores con `offset-x` en % | Adoptado como **sistema**: cada indicador es su propia ventana X esparcida por el borde superior |
| `colors.ini` con `include-file` | Mantenido; la paleta se extendió (blue-gray, white, trans) y conviven hex hardcodeados (`#81a1c1` Nord) |
| `launch.sh` (killall → wait loop → launch) | Mantenido casi idéntico; solo cambia la lista de barras lanzadas y la mezcla de `current.ini` + `workspace.ini` |
| Fuente Iosevka Nerd Font | Mantenida como `font-0` + 7 fuentes más (Montserrat, Hurmit, Hack) |
| Módulos del tema (workspaces, clock, powermenu) | Ampliados con los indicadores de laboratorio (`custom/script`: ethernet/VPN/target) y `settarget`/`cleartarget` |
| Módulo workspaces con iconos | Reconvertido: `icon-0…4` mapean 1→`-` y 2–5 a glifo vacío, con colores Nord por estado |

Lectura práctica: cuando algo de la config personal no se entienda, buscar su contraparte en el tema original — suele ser la versión "base" de la misma idea, más comentada en el repo upstream.

## Flujo de trabajo: aplicar/evolucionar el tema

```bash
# 1. respaldo siempre primero
cp -r ~/.config/polybar ~/.config/polybar.bak-$(date +%F)

# 2. aplicar el tema
cd ~/Descargas/blue-sky/polybar && cp -r * ~/.config/polybar

# 3. fuentes si es una máquina nueva
sudo cp fonts/* /usr/share/fonts/truetype && fc-cache -v

# 4. relanzar barras (o bspc wm -r para toda la sesión)
killall -q polybar; ~/.config/polybar/launch.sh
```

Para **personalizar** el tema en vez de adoptarlo tal cual (el camino que siguió este vault): editar `colors.ini` para la paleta propia, partir de los bloques `[bar/*]` como plantilla y añadir módulos — la referencia completa de módulos y tags está en [[Configuración de Polybar]].

## Errores comunes

1. **Barras no aparecen tras instalar**: el `launch.sh` no es ejecutable (`chmod +x ~/.config/polybar/launch.sh`) o `bspwmrc` no se ha re-ejecutado (`bspc wm -r`).
2. **Iconos/letras en caja**: faltan las fuentes o `fc-cache` no corrió — `fc-list | grep -i iosevka`.
3. **Sobrescritura del setup personal**: `cp -r *` pisó la config — por eso el respaldo del flujo de trabajo.
4. **Transparente opaco**: picom no está corriendo — `pgrep -a picom`.
5. **Nombre duplicado de barra**: el `launch.sh` del tema lanza barras por nombre; si la config fue reemplazada, los nombres ya no existen → leer el error `Undefined bar name` del log.
6. **Barras pegadas al borde (perdió el look flotante)**: el tema original usaba `override-redirect = true` y bspwm re-apila con `wm-restack`; probar una de las dos vías según el desired stacking.
7. **`include-file` roto tras mezclar temas**: un `config.ini` copiado de otro tema puede incluir su propio `colors.ini` y pisar la paleta — revisar el `include-file` efectivo (el log lo reporta si el archivo no existe).

## Cheatsheet del tema

| Paso | Comando |
|---|---|
| Clonar | `git clone https://github.com/VaughnValle/blue-sky.git` |
| Instalar config | `cp -r blue-sky/polybar/* ~/.config/polybar` |
| Fuentes | `sudo cp blue-sky/polybar/fonts/* /usr/share/fonts/truetype && fc-cache -v` |
| Autostart | `echo '~/.config/polybar/./launch.sh &' >> ~/.config/bspwm/bspwmrc` |
| Relanzar barras | `killall -q polybar; ~/.config/polybar/launch.sh` |
| Probar en foreground | `polybar main -c ~/.config/polybar/config.ini -l info` |
| Cambiar acento | editar `colors.ini` → `${color.ac}` |
| Respaldar antes | `cp -r ~/.config/polybar ~/.config/polybar.bak-$(date +%F)` |

## Relacionadas

- [[Configuración de Polybar]] — la configuración personal que evolucionó de este tema
- [[Instalación de Polybar y Picom]] — instalación del binario y de picom
- [[Configuración de Picom]] — esquinas redondeadas y transparencia que hacen honor al tema
- [[Configuración de BSPWM y SXHKD]] — el `bspwmrc` donde se enlaza `launch.sh`
- [[Configuración Post-Instalación de Entorno de Escritorio]] — fuentes (Nerd Fonts) a nivel sistema/usuario
