----
Ya con el  [[Configurar - Fuente, Kitty, Wallpaper]] listo vamos a ejecutar el siguiente comando dentro de descargas:

```bash
git clone https://github.com/VaughnValle/blue-sky.git
cd blue-sky/polybar/ 
cp -r * ~/.config/polybar
echo '~/.config/polybar/./launch.sh &' >> ~/.config/bspwm/bspwmrc
sudo cp fonts/* /usr/share/fonts/truetype
fc-cache -v
```
