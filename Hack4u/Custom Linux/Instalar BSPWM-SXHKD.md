----

En resumen, primeramente ejecutaremos el siguiente comando:

```Bash
sudo apt install build-essential git vim xcb libxcb-util0-dev libxcb-ewmh-dev libxcb-randr0-dev libxcb-icccm4-dev libxcb-keysyms1-dev libxcb-xinerama0-dev libasound2-dev libxcb-xtest0-dev libxcb-shape0-dev
```

Posteriormente, aplicaremos una actualización del sistema con el comando ‘**apt update**‘. Acto seguido, tenéis que dirigiros a la carpeta de descargas de vuestro equipo y descargar los proyectos ‘**bswpm**‘ y ‘**sxhkd**‘ con los siguientes comandos:

```Bash
git clone https://github.com/baskerville/bspwm.git
git clone https://github.com/baskerville/sxhkd.git
```

Para instalar cada uno de estos, lo que debéis hacer es meteros en ambos directorios por separado y ejecutar los comandos ‘**make**‘ y ‘**sudo make install**‘.

Luego de esto comenzamos a configurar los archivos propios tanto de sxhkd y bspwm, para esto ejecutamos los siguientes comandos:

```bash
mkdir ~/.config/{bspwm,sxhkd}
```

luego de esto ejecutamos el siguiente comando:

```bash
cp ~/Descargas/bspwm/examples/bspwmrc ~/.config/bspwm/ && cp ~/Descargas/bspwm/examples/sxhkdrc ~/.config/sxhkd/
```

Vamos a seguir instalando algunos paquetes como son:

```bash
sudo apt install kitty rofi imagemagick feh wmname
```

La configuración final del archivo sxhkdrc es la siguiente:

```bash
#
# wm independent hotkeys
#

# terminal emulator
super + Return
	/usr/bin/kitty

# program launcher
super + d
	/usr/bin/rofi -show run

# make sxhkd reload its configuration files:
super + Escape
	pkill -USR1 -x sxhkd

#
# bspwm hotkeys
#

# quit/restart bspwm
super + shift + {q,r}
	bspc {quit,wm -r}

# close and kill
super + {_,shift + }q
	bspc node -{c,k}

# alternate between the tiled and monocle layout
super + m
	bspc desktop -l next

# send the newest marked node to the newest preselected node
super + y
	bspc node newest.marked.local -n newest.!automatic.local

# swap the current node and the biggest window
super + g
	bspc node -s biggest.window

#
# state/flags
#

# set the window state
super + {t,shift + t,s,f}
	bspc node -t {tiled,pseudo_tiled,floating,fullscreen}

# set the node flags
super + ctrl + {m,x,y,z}
	bspc node -g {marked,locked,sticky,private}

#
# focus/swap
#

# focus the node in the given direction
super + {_,shift + }{Left,Down,Up,Right}
	bspc node -{f,s} {west,south,north,east}

# focus the node for the given path jump
super + {p,b,comma,period}
	bspc node -f @{parent,brother,first,second}

# focus the next/previous window in the current desktop
super + {_,shift + }c
	bspc node -f {next,prev}.local.!hidden.window

# focus the next/previous desktop in the current monitor
super + bracket{left,right}
	bspc desktop -f {prev,next}.local

# focus the last node/desktop
super + {grave,Tab}
	bspc {node,desktop} -f last

# focus the older or newer node in the focus history
super + {o,i}
	bspc wm -h off; \\
	bspc node {older,newer} -f; \\
	bspc wm -h on

# focus or send to the given desktop
super + {_,shift + }{1-9,0}
	bspc {desktop -f,node -d} '^{1-9,10}'

#
# preselect
#

# preselect the direction
super + ctrl + alt + {Left,Down,Up,Right}
	bspc node -p {west,south,north,east}

# preselect the ratio
super + ctrl + {1-9}
	bspc node -o 0.{1-9}

# cancel the preselection for the focused node
super + ctrl + alt + space
	bspc node -p cancel

# cancel the preselection for the focused desktop
super + ctrl + shift + space
	bspc query -N -d | xargs -I id -n 1 bspc node id -p cancel

#
# move/resize
#

# move a floating window
super + shift + {Left,Down,Up,Right}
	bspc node -v {-20 0,0 20,0 -20,20 0}

# Custom resize

super + alt + {Left,Down,Up,Right}
	/home/stark/.config/bspwm/scripts/bspwm_resize {west,south,north,east}

```

A continuación tenéis el contenido del archivo de configuración ‘**bspwm_resize**‘ que usamos al final de esta clase:

```bash
#!/usr/bin/env dash

if bspc query -N -n focused.floating > /dev/null; then
	step=20
else
	step=100
fi

case "$1" in
	west) dir=right; falldir=left; x="-$step"; y=0;;
	east) dir=right; falldir=left; x="$step"; y=0;;
	north) dir=top; falldir=bottom; x=0; y="-$step";;
	south) dir=top; falldir=bottom; x=0; y="$step";;
esac

bspc node -z "$dir" "$x" "$y" || bspc node -z "$falldir" "$x" "$y"
```

El archivo lo vamos a crear en la carpeta .config/bspwm/scripts/ y el contenido es el de arriba.