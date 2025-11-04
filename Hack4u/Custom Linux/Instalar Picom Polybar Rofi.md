bs----

Ya con [[Instalar BSPWM-SXHKD]] listo vamos a realizar la instalación del a siguiente forma:

- Para instalar la Polybar vamos a hacer:

```bash
sudo apt install polybar
```

- Para instalar picom vamos a realizar los siguientes comandos dentro de la carpeta descargas:

```bash
sudo apt install libconfig-dev libdbus-1-dev libegl-dev libev-dev libgl-dev libepoxy-dev libpcre2-dev libpixman-1-dev libx11-xcb-dev libxcb1-dev libxcb-composite0-dev libxcb-damage0-dev libxcb-dpms0-dev libxcb-glx0-dev libxcb-image0-dev libxcb-present-dev libxcb-randr0-dev libxcb-render0-dev libxcb-render-util0-dev libxcb-shape0-dev libxcb-util-dev libxcb-xfixes0-dev libxext-dev meson ninja-build uthash-dev

git clone https://github.com/yshui/picom
cd picom

meson setup --buildtype=release build
ninja -C build
ninja -C build install
```

Ya con todo esto vamos a escribir el siguiente comando el cual va a cerrar la sesión: