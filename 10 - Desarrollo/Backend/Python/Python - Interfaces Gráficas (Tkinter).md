---
tags:
  - desarrollo
  - backend
  - python
tipo: concepto
estado: terminado
---

# Python - Interfaces Gráficas (Tkinter)

Referencia profunda de **Tkinter**, la biblioteca de GUI de la stdlib de Python: anatomía de la aplicación y el bucle de eventos, los tres gestores de geometría en detalle, catálogo completo de widgets, variables de enlace, eventos, ttk y temas, diálogo con hilos, y tres proyectos completos (editor de texto, calculadora, chat multiusuario).

> [!info] Qué es Tkinter y cómo se apila
> Tkinter es un **wrapper de Python para Tk**, el toolkit gráfico escrito en C/Tcl (creado en 1991 para el editor de Tcl). La pila es: tu código Python → módulos `tkinter` (Python) → Tcl/Tk (C) → X11 / Win32 / Aqua del sistema. Por eso el look y feel es nativo-de-plataforma-ish y por eso el **bucle de eventos** es Tcl: `mainloop()` le entrega el control a Tcl, que despacha los eventos del SO de vuelta a callbacks Python. `import tkinter` levanta todo ese stack — en Linux requiere `libtk8.6`/`tk-dev` instaladas (en esta máquina el import falla con `ImportError: libtk8.6.so: cannot open shared object file` si falta el paquete del SO).

**Por qué Tkinter**: viene con Python (cero dependencias), es portable (Windows/macOS/Linux sin tocar código), ideal para herramientas internas, prototipos, CLIs con cara, educación. **Cuándo NO**: apps comerciales con look moderno, mobile, o UIs con muchos datos → ahí PySide6/Qt (ver "Cuándo elegir Qt" al final).

> [!warning] Tcl/Tk 8.6 y `mainloop`
> Sin `mainloop()` la ventana se dibuja (un solo update) y el programa termina: no hay repaint, no hay clics. Es lo que mantiene viva la aplicación — un bucle infinito que espera eventos (ratón, teclado, resize, timers `after`) y ejecuta sus callbacks.

## Anatomía de una aplicación

```python
#!/usr/bin/env python3
import tkinter as tk

def accion_boton():
    print("[+] Se ha presionado el botón")

root = tk.Tk()                     # crea la ventana raíz + inicializa Tcl/Tk (singleton)
root.title("Mi primera aplicación")
root.geometry("640x480+100+80")    # ancho x alto +X +Y (posición en pantalla)
root.minsize(320, 240)

label = tk.Label(root, text="¡Hola Mundo!", font=("Arial", 16))
label.pack()                       # sin pack/grid/place NO se ve

button = tk.Button(root, text="Presióname", command=accion_boton)
button.pack()

root.mainloop()                    # entrega el control a Tcl: todo lo anterior es setup
```

- `tk.Tk()` es la **raíz**: hay una sola. Los `Toplevel` son ventanas hijas.
- Widgets **no existen visualmente** hasta que un gestor de geometría los gestiona — "crear" es instanciar el objeto.
- `command=accion_boton` recibe la función **sin llamarla** (sin paréntesis). Para pasar argumentos: `command=lambda: accion_boton(x)` o `functools.partial`.

### Ciclo de vida y métodos de la raíz

| Método | Función |
|---|---|
| `mainloop(n=0)` | Bucle de eventos; devuelve al cerrar la ventana |
| `quit()` | Sale del mainloop **sin** destruir (la ventana queda) |
| `destroy()` | Destruye la ventana y sus hijos (y termina el mainloop si es la raíz) |
| `title(t)`, `iconname()` | Título e icono |
| `geometry("WxH+X+Y")` | Tamaño/posición iniciales |
| `resizable(w, h)` | Si se puede redimensionar |
| `config(**opts)` | Cualquier opción de la raíz (bg, padx, cursor...) |
| `after(ms, fn)` / `after_cancel(id)` | **Timer dentro del event loop** — la base de animaciones y tareas periódicas |
| `update()`, `update_idletasks()` | Forzar redibujado (usar con cuidado; rompe el flujo de eventos) |

`root.after(1000, fn)` es el reloj de Tk: ejecuta `fn` dentro del bucle de eventos cada 1 s **sin bloquear** la interfaz. Patrón de tick: `def tick(): label.config(text=time.strftime("%H:%M")); root.after(1000, tick)`.

## Gestores de geometría en profundidad

Tkinter tiene **tres** gestores; son **mutuamente excluyentes dentro del mismo contenedor**: mezclar `pack()` y `grid()` sobre los hijos del mismo frame lanza `TclError: cannot use geometry manager pack inside .!frame which already has slaves managed by grid`. Lo que sí se puede (y es el patrón correcto) es anidar gestores distintos en contenedores distintos: `pack` en la raíz para una toolbar y un `grid` dentro del frame del formulario.

### `pack()` — bloques apilados (el rápido)

Organiza hijos en bordes, en orden de llamada.

| Opción | Valores | Efecto |
|---|---|---|
| `side` | `TOP` (defecto), `BOTTOM`, `LEFT`, `RIGHT` | Borde donde se apila |
| `fill` | `X`, `Y`, `BOTH`, `NONE` | Estirar hasta llenar el espacio que le queda **en la dirección asignada** |
| `expand` | `True/False` | Si True, el widget se lleva el espacio sobrante del contenedor (y `fill` lo estira) |
| `padx/pady`, `ipadx/ipady` | px | Padding externo / interno |
| `anchor` | `n, s, e, w, center...` | Alineación dentro de su caja |

```python
label1 = tk.Label(root, text="Mi primer label", bg="red")
label2 = tk.Label(root, text="Mi segundo label", bg="green")
label3 = tk.Label(root, text="Mi tercer label", bg="blue")

label1.pack(fill=tk.X)                    # banda horizontal completa
label2.pack(fill=tk.X)
label3.pack(side=tk.LEFT, fill=tk.Y)      # columna vertical a la izquierda
```

Regla práctica: **`side` define el orden de empaquetado; `fill` estira dentro del hueco que quedó; `expand` reparte el sobrante.** La combinación "barra de herramientas arriba, barra de estado abajo, contenido expandible al medio" es:

```python
toolbar.pack(side=tk.TOP, fill=tk.X)
statusbar.pack(side=tk.BOTTOM, fill=tk.X)
contenido.pack(side=tk.TOP, fill=tk.BOTH, expand=True)   # se queda con todo lo demás
```

### `grid()` — cuadrícula (el de formularios)

| Opción | Efecto |
|---|---|
| `row`, `column` | Celda (0-based; column puede omitirse) |
| `rowspan`, `columnspan` | Celdas combinadas |
| `sticky` | `N S E W` combinadas: estira/alinea dentro de su celda (como `fill`+`anchor` en uno) |
| `weight` (por fila/columna) | **Quién se lleva el espacio al redimensionar** — sin pesos, todo el sobrante va a la última celda |
| `padx/pady`, `ipadx/ipady` | Padding |
| `rowconfigure(c, weight=1)` | El gestor **del contenedor**: "esta fila crece" |

```python
label1 = tk.Label(root, text="Mi primer label", bg="red")
label2 = tk.Label(root, text="Mi segundo label", bg="green")
label3 = tk.Label(root, text="Mi tercer label", bg="blue")

label1.grid(row=0, column=0)
label2.grid(row=0, column=1)
label3.grid(row=1, column=0, columnspan=2, sticky="ew")

root.columnconfigure(0, weight=1)   # columna 0 crece con la ventana
root.columnconfigure(1, weight=3)   # la 1 crece 3 veces más rápido
```

> [!tip] `weight` es la propagación de Tkinter
> Equivale a `flex-grow`/`grid-template-columns: 1fr` de CSS (ver [[Flexbox]]): sin `weight=1`, al agrandar la ventana **ninguna** columna crece y todo se queda arriba a la izquierda. Casi todo bug de "mi layout no escala" es un `weight` faltante.

### `place()` — posición absoluta/relativa (el peligroso)

| Opción | Efecto |
|---|---|
| `x`, `y` | Coordenadas absolutas en px desde la esquina del padre |
| `relx`, `rely`, `relwidth`, `relheight` | Fracción 0-1 del tamaño del padre (responsive) |
| `width`, `height` | Tamaño absoluto |
| `anchor` | Punto del widget que cae en las coordenadas (`center`, `nw`...) |

```python
label1.place(x=20, y=20)                            # 20px desde arriba-izquierda
label2.place(relx=0.8, rely=0.2)                    # 80%/20% del área
label3.place(relx=0.5, rely=0.5, anchor=tk.CENTER)  # centrado sin importar el tamaño
```

> [!warning] `place` rompe layouts responsivas
> Posiciones absolutas no escalan con el contenido ni con la ventana. Úsalo solo para: overlays puntuales (tooltips, badges, popups sobre un widget), o combinado con `relx/rely` para centrados. Para el layout general, `grid` (formularios) o `pack` (bandas).

### Elegir gestor

| Situación | Gestor |
|---|---|
| Formulario / tabla / settings | `grid` |
| Banda arriba/abajo/izquierda (toolbar, sidebar, statusbar) | `pack` |
| Overlay, watermark, centrado pixel-perfect | `place` (con `rel*`) |
| Layout complejo | Anidamiento: `pack` en la raíz, `grid` dentro de cada frame |

## Catálogo de widgets

### `Label` — texto/imagen estático

Opciones clave: `text`, `font=("Arial", 12, "bold")`, `fg`/`bg` (color de texto/fondo), `image` (ver PIL más abajo), `compound="left"` (imagen + texto), `wraplength=300` (parte el texto largo), `justify="center"`, `anchor`. Actualización: `label.config(text="nuevo")` o vía `StringVar`.

### `Button`

`text`, `command`, `state="disabled"`, `bg`, `activebackground`, `width` (en caracteres). El patrón "botón que cambia" usa dos comandos o un lambda que alterna.

### `Entry` — una línea de texto

```python
entrada = tk.Entry(root, show="*", width=30)   # show="*" para contraseñas
entrada.insert(0, "texto inicial")
valor = entrada.get()                          # str
entrada.delete(0, tk.END)                      # limpiar
entrada.config(state="readonly")               # ver sin editar
```

Índices: `0`/`tk.END`; también `insert("end", ...)`. `Entry` no tiene `get` reactiva: el patrón es leer en el evento (botón/Enter).

### `Text` — múltiples líneas

```python
texto = tk.Text(root, wrap="word", height=15, state="disabled")
texto.insert("1.0", "Primera línea\nSegunda línea")   # índice "fila.columna" (1-based)
contenido = texto.get("1.0", "end-1c")                # -1c: sin el \n fantasma final
texto.delete("1.0", tk.END)
texto.config(state="normal")   # escribir; volver a "disabled" para solo lectura
texto.tag_add("sel", "1.0", "1.5")                    # selección por rangos
texto.tag_config("err", foreground="red")             # estilos por tag
texto.see("end")                                      # autoscroll
```

El índice `"1.0"` = fila 1, columna 0. `"end-1c"` es el truco para obtener el contenido sin el newline que Text agrega siempre.

### `Listbox`, `Scrollbar` y el vínculo con `Text`/`Canvas`

```python
lista = tk.Listbox(root, selectmode=tk.MULTIPLE)  # SINGLE/BROWSE/MULTIPLE/EXTENDED
for item in ["uno", "dos", "tres"]:
    lista.insert(tk.END, item)
seleccion = lista.curselection()     # tupla de índices
lista.delete(0, tk.END)              # vaciar (índice 0 a END)
lista.itemconfig(0, bg="yellow")

scroll = tk.Scrollbar(root, orient=tk.VERTICAL, command=lista.yview)
lista.config(yscrollcommand=scroll.set)   # doble enlace: scrollbar ↔ widget
```

El doble enlace (`yscrollcommand`/`command`) es manual en tk clásico; en ttk existe `ttk.Scrollbar` con el mismo patrón. `ScrolledText` (`tkinter.scrolledtext`) ya viene con su scrollbar integrada.

### `Frame` — contenedores invisibles

Agrupan widgets para layout y estilo (`bg`, `bd=5`, `relief=tk.RIDGE`). Es la unidad de anidamiento de gestores. `LabelFrame` añade título dibujado alrededor del grupo.

### `Canvas` — dibujo vectorial

```python
canvas = tk.Canvas(root, width=500, height=500, bg="white")
canvas.pack(pady=20)
oval = canvas.create_oval(50, 50, 150, 150, fill="red", outline="")
rect = canvas.create_rectangle(200, 50, 350, 100, fill="green")
line = canvas.create_line(50, 250, 200, 320, fill="blue", width=5)
texto_id = canvas.create_text(100, 300, text="hola", fill="black")
canvas.move(oval, 10, 0)                     # desplazar
canvas.coords(oval)                          # bounding box actual
canvas.itemconfig(rect, fill="blue")         # recolorear
canvas.delete("all")
canvas.create_window(250, 250, widget=mi_frame)  # embeber widgets en el canvas
```

### Selección simple: `Radiobutton`, `Checkbutton`, `Scale`

```python
genero = tk.StringVar(value="f")
tk.Radiobutton(root, text="Femenino", variable=genero, value="f").pack()
tk.Radiobutton(root, text="Masculino", variable=genero, value="m").pack()

aviso = tk.IntVar(value=1)
tk.Checkbutton(root, text="Avisos", variable=aviso).pack()   # 0/1

volumen = tk.IntVar()
tk.Scale(root, from_=0, to=100, orient=tk.HORIZONTAL,
         variable=volumen, command=lambda v: print(v)).pack()
```

### `Toplevel` — ventanas secundarias

```python
def abrir_acerca():
    top = tk.Toplevel(root)
    top.title("Acerca de")
    tk.Label(top, text="v1.0").pack(padx=20, pady=20)
    top.transient(root)   # siempre sobre la raíz
    top.grab_set()        # modal: bloquea la raíz hasta cerrar
```

### `Menu` — barra de menús y contextuales

```python
barra = tk.Menu(root)
root.config(menu=barra)

menu_archivo = tk.Menu(barra, tearoff=0)   # tearoff=0 elimina la línea de "desprendible"
barra.add_cascade(label="Archivo", menu=menu_archivo)
menu_archivo.add_command(label="Nuevo", command=nuevo, accelerator="Ctrl+N")
menu_archivo.add_separator()
menu_archivo.add_checkbutton(label="Modo oscuro", variable=oscuro)
menu_archivo.add_radiobutton(label="Español", variable=idioma, value="es")
menu_archivo.add_command(label="Salir", command=root.destroy)

# Menú contextual con clic derecho:
menu_contexto = tk.Menu(root, tearoff=0)
menu_contexto.add_command(label="Copiar")
root.bind("<Button-3>", lambda e: menu_contexto.tk_popup(e.x_root, e.y_root))
```

## Variables de enlace: `StringVar`, `IntVar`, `DoubleVar`, `BooleanVar`

Son **proxy observables** entre el mundo Tk y el Python: Tk las lee/escribe vía Tcl, y tu código recibe notificación con `.trace_add("write", callback)`:

```python
nombre = tk.StringVar(value="inicial")
entrada = tk.Entry(root, textvariable=nombre)   # el Entry queda espejado
label = tk.Label(root, textvariable=nombre)     # el Label también

def al_escribir(*_):
    print("ahora dice:", nombre.get())
nombre.trace_add("write", al_escribir)          # callback con (var, index, op)
```

En tk antiguo era `trace("w", fn)`; usar `trace_add` (3.6+). Ventaja sobre `entry.get()`: el estado vive en un solo sitio y muchos widgets pueden compartirlo.

## Eventos: `bind` y sintaxis de patrones

| Patrón | Evento |
|---|---|
| `<Button-1>` / `<Button-3>` | Clic izquierdo / derecho |
| `<Double-Button-1>` | Doble clic |
| `<B1-Motion>` | Arrastrar con botón 1 |
| `<Motion>` | Movimiento del ratón |
| `<Key>` / `<KeyPress-a>` | Cualquier tecla / la tecla `a` |
| `<Return>`, `<Escape>`, `<BackSpace>` | Teclas nombradas |
| `<Control-c>` | Ctrl+C (también `<Control-KeyPress-c>`) |
| `<FocusIn>` / `<FocusOut>` | Entra/sale el foco |
| `<Enter>` / `<Leave>` | Ratón entra/sale **el widget** (no confundir con `<Return>`) |

```python
def on_key(event):
    print(event.char, event.keycode, event.keysym)

entrada.bind("<Key>", on_key)
entrada.bind("<Return>", lambda e: enviar())     # el handler recibe SIEMPRE el evento
canvas.bind("<Button-1>", lambda e: canvas.create_oval(e.x-5, e.y-5, e.x+5, e.y+5))

root.bind_all("<Control-n>", nuevo)              # hereda a todos los widgets
root.bind("<Configure>", lambda e: print("resize"))   # redimensionamiento
```

**Virtual events** (`event_generate`): eventos de alto nivel que defines vos:

```python
root.event_generate("<<Guardado>>")
root.bind("<<Guardado>>", lambda e: status.config(text="Guardado"))
```

Los `<<...>>` entre corchetes dobles son la convención ttk/tema (p.ej. `<<ComboboxSelected>>`, `<<TreeviewSelect>>`).

## ttk: los widgets modernos y los temas

`ttk` (Tk 8.5+) son widgets con **apariencia themable** — en macOS/Windows usan los controles nativos. Regla práctica: **prefiere ttk**; tk clásico solo cuando necesites sus opciones de estilo directo (`bg` crudo).

| tk clásico | ttk equivalente | Diferencia |
|---|---|---|
| `Label` | `ttk.Label` | Estilo themable; sin `bg` directo (usar `style`) |
| `Button` | `ttk.Button` | — |
| `Entry` | `ttk.Entry` | — |
| `Checkbutton`/`Radiobutton` | `ttk.Checkbutton`/`ttk.Radiobutton` | Trabajan con `variable` igual |
| `Scale` | `ttk.Scale` | — |
| `Combobox` (no existe en tk) | `ttk.Combobox` | Desplegable de texto; `values=[...]`, `state="readonly"` |
| — | `ttk.Treeview` | Tabla/jerarquía (el widget de datos serio) |
| — | `ttk.Notebook` | Pestañas |
| — | `ttk.Separator`, `ttk.Progressbar` | Separadores y progreso |
| `Scrollbar` | `ttk.Scrollbar` | — |

```python
from tkinter import ttk

combo = ttk.Combobox(root, values=["Rojo", "Verde", "Azul"], state="readonly")
combo.current(0)                                    # selecciona índice 0
combo.bind("<<ComboboxSelected>>", lambda e: print(combo.get()))

nb = ttk.Notebook(root)
pestana1 = ttk.Frame(nb)
nb.add(pestana1, text="Configuración")
nb.pack(fill="both", expand=True)

tree = ttk.Treeview(root, columns=("precio",), show="tree headings")
tree.heading("#0", text="Producto"); tree.heading("precio", text="Precio")
tree.insert("", "end", text="Café", values=("3.50",))

prog = ttk.Progressbar(root, maximum=100)
prog.start(10)   # animación indeterminada cada 10ms
```

### Temas

```python
estilo = ttk.Style()
print(estilo.theme_use())            # tema activo (p.ej. 'clam' en Linux)
print(estilo.theme_names())          # ('clam', 'alt', 'default', 'classic'...)
estilo.theme_use("clam")

# Personalizar un elemento:
estilo.configure("TButton", padding=6, font=("Arial", 11))
estilo.map("TButton", background=[("active", "#4a6fd4")])
boton = ttk.Button(root, text="OK", style="TButton")
```

## Layout patrón: toolbar + statusbar + contenido

```python
class App:
    def __init__(self, root):
        self.root = root
        root.title("App completa")

        toolbar = ttk.Frame(root); toolbar.pack(side="top", fill="x")
        ttk.Button(toolbar, text="Abrir").pack(side="left")
        ttk.Button(toolbar, text="Guardar").pack(side="left")

        self.contenido = tk.Text(root); self.contenido.pack(side="top", fill="both", expand=True)
        # ↑ expand=True: el contenido se lleva el espacio sobrante

        self.status = tk.Label(root, text="Listo", anchor="w", bd=1, relief="sunken")
        self.status.pack(side="bottom", fill="x")     # el último pack BOTTOM queda abajo
```

### Frame con scroll (receta clásica)

Tk no tiene scroll nativo para un contenedor; se compone `Canvas` + `Frame` interno + `Scrollbar`, sincronizando el scroll del canvas con la geometría del frame:

```python
def crear_frame_scroll(padre):
    cont = ttk.Frame(padre)
    canvas = tk.Canvas(cont, highlightthickness=0)
    sb = ttk.Scrollbar(cont, orient="vertical", command=canvas.yview)
    interno = ttk.Frame(canvas)

    interno.bind("<Configure>",
        lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
    win = canvas.create_window((0, 0), window=interno, anchor="nw")
    canvas.configure(yscrollcommand=sb.set)
    # que el frame interno siga el ancho del canvas:
    canvas.bind("<Configure>", lambda e: canvas.itemconfig(win, width=e.width))

    canvas.pack(side="left", fill="both", expand=True)
    sb.pack(side="right", fill="y")
    return cont, interno
```

## Diálogos: `messagebox` y `filedialog`

No son widgets: son módulos con funciones que invocan los diálogos **del sistema operativo** (look nativo, comportamiento modal conocido).

```python
from tkinter import messagebox, filedialog, colorchooser, simpledialog

messagebox.showinfo("Título", "Mensaje")
messagebox.showwarning("Cuidado", "Algo raro")
messagebox.showerror("Error", "Algo roto")
ok = messagebox.askokcancel("Salir", "¿Cerrar?")            # True/False
seguro = messagebox.askyesno("Eliminar", "¿Seguro?")        # True/False
respuesta = messagebox.askyesnocancel("Guardar", "¿Guardar cambios?")  # True/False/None

ruta = filedialog.askopenfilename(
    title="Abrir", filetypes=[("Texto", "*.txt"), ("Todos", "*.*")])
ruta_save = filedialog.asksaveasfilename(defaultextension=".txt")
carpeta = filedialog.askdirectory()

color = colorchooser.askcolor(title="Elegir color")          # ((r,g,b), '#rrggbb')
nombre = simpledialog.askstring("Nombre", "¿Cómo te llamás?")
```

`askopenfilename` devuelve `""` si el usuario cancela — **siempre** chequear antes de abrir el archivo.

## Hilos y `mainloop`: por qué no podés tocar la GUI desde otro hilo

Tk **no es thread-safe**: los widgets solo pueden tocarse desde el hilo que ejecutó `mainloop()`. Los síntomas de violarlo son los peores bugs que existen: crashes intermitentes en `Tcl_DoOneEvent`, ventanas que se congelan, o excepciones `RuntimeError: main thread is not in main loop` — nada confiable.

El patrón correcto: **hilo de trabajo + `queue.Queue` + timer `after`** que drena la cola en el hilo principal (misma receta que el chat del Proyecto 3):

```python
import queue
import threading

cola: queue.Queue = queue.Queue()

def trabajador():
    resultado = trabajo_largo()          # minutos de CPU/IO, en su hilo
    cola.put(("listo", resultado))       # nunca tocar widgets aquí

def drenar():
    try:
        while True:
            tipo, dato = cola.get_nowait()
            if tipo == "listo":
                status.config(text=f"Terminado: {dato}")
    except queue.Empty:
        pass
    root.after(100, drenar)              # re-programar el sondeo (100ms)

threading.Thread(target=trabajador, daemon=True).start()
root.after(100, drenar)
root.mainloop()
```

Variante sin sondeo: `root.after_idle(fn)` o la librería `pysimplegui`-style; también existe `root.event_generate("<<Resultado>>", ...)` combinado con `bind`. Lo que **nunca** funciona: `label.config(...)` desde el hilo worker.

## Imágenes: PIL/Pillow

Tk soporta nativamente **GIF y PNG** (`tk.PhotoImage(file=...)`); para JPEG u operaciones hay que usar Pillow:

```python
from PIL import Image, ImageTk

img = Image.open("foto.jpg").resize((200, 150))
foto = ImageTk.PhotoImage(img)          # ¡guardar referencia! ver warning
label = tk.Label(root, image=foto)
```

> [!warning] La referencia de la imagen desaparece y el label queda en blanco
> `PhotoImage` es un wrapper de objetos Tcl: si no guardás la referencia (`self.foto = ImageTk.PhotoImage(...)` en la clase), el recolector de basura la libera y el widget muestra **nada**. Es el bug #1 de imágenes en Tkinter. Siempre atributo de instancia.

## Empaquetado para distribución

Tkinter corre donde esté Python, pero el usuario final no tiene Python: **PyInstaller** genera el ejecutable:

```bash
pip install pyinstaller
pyinstaller --onefile --windowed --name MiApp main.py   # --windowed: sin consola
# salida: dist/MiApp (exe/macOS app); ver [[Python - Módulos, Paquetes y Distribución]]
```

Alternativas: `briefcase` (BeeWare, más control), `Nuitka` (compila a C).

## Cuándo elegir Qt (PySide6) en su lugar

| Criterio | Tkinter | PySide6 / PyQt6 |
|---|---|---|
| Instalación | Con Python | `pip install pyside6` (~100 MB) |
| Look & feel | Genérico (mejor con ttk) | Nativo/Qt, moderno |
| Widgets de datos | `Treeview` básico | Tablas, árboles, modelos MVC completos |
| Diseñador visual | No hay | Qt Designer |
| Rendimiento de UI pesada | Tcl intermedio | C++ por debajo |
| Licencia | Python | LGPL (PySide) / GPL-comercial (PyQt) |
| Curva de aprendizaje | Baja | Media |

Elegí Tkinter para: herramientas internas, CLIs con cara, educación, apps de una ventana. Elegí PySide para: apps comerciales, editores, UIs con muchas tablas/gráficos, si el cliente exige look nativo pulido.

## Proyecto 1 — Editor de texto (blog de notas)

Aplica: `Text`, `Menu`, `filedialog`, `messagebox`, POO.

```python
#!/usr/bin/env python3
import tkinter as tk
from tkinter import filedialog, messagebox

class SimpleTextEditor:
    def __init__(self, root):
        self.root = root
        self.text_area = tk.Text(self.root)
        self.text_area.pack(fill=tk.BOTH, expand=1)   # ocupa toda la ventana
        self.current_open_file = ""

    def quit_confirm(self):
        if messagebox.askokcancel("Salir", "¿Estás seguro que deseas salir?"):
            self.root.destroy()

    def open_file(self):
        filename = filedialog.askopenfilename()       # ruta absoluta o ""
        if filename:
            if self.text_area.get("1.0", tk.END).strip():
                opt = messagebox.askyesno(
                    "Eliminar", "Tienes información sin guardar. ¿Deseas eliminarla?")
                if not opt:
                    return
            self.text_area.delete("1.0", tk.END)
            with open(filename, "r", encoding="utf-8") as f:
                self.text_area.insert("1.0", f.read())
            self.current_open_file = filename

    def new_file(self):
        if self.text_area.get("1.0", tk.END).strip():
            if not messagebox.askyesno("Eliminar", "¿Crear archivo nuevo?"):
                return
        self.text_area.delete("1.0", tk.END)
        self.current_open_file = ""

    def save_file(self):
        if not self.current_open_file:
            filename = filedialog.asksaveasfilename()
            if not filename:
                return
            self.current_open_file = filename
        with open(self.current_open_file, "w", encoding="utf-8") as f:
            f.write(self.text_area.get("1.0", "end-1c"))

root = tk.Tk()
root.title("Editor de Notas")
editor = SimpleTextEditor(root)

menu_bar = tk.Menu(root)
menu_options = tk.Menu(menu_bar, tearoff=0)
menu_options.add_command(label="Nuevo", command=editor.new_file)
menu_options.add_command(label="Abrir", command=editor.open_file)
menu_options.add_command(label="Guardar", command=editor.save_file)
menu_options.add_command(label="Salir", command=editor.quit_confirm)
root.config(menu=menu_bar)
menu_bar.add_cascade(label="Archivo", menu=menu_options)

root.mainloop()
```

Detalles que agregan calidad (respecto a la versión mínima del curso): `encoding="utf-8"` explícito, `.strip()` para no preguntar "guardar cambios" con un archivo vacío que solo tiene el `\n` fantasma de `Text`, y `"end-1c"` para no guardar un newline extra.

## Proyecto 2 — Calculadora con teclado

Aplica: `Entry` como display, `grid` + `columnspan`, `bind("<Key>")`, lambdas.

```python
#!/usr/bin/env python3
import tkinter as tk

class Calculadora:
    def __init__(self, master):
        self.master = master
        self.display = tk.Entry(master, width=15, font=("Arial", 23), bd=10,
                                insertwidth=1, bg="#6495DE", justify="right")
        self.display.grid(row=0, column=0, columnspan=4, padx=10, pady=10)
        self.op_verification = False
        self.current = ""
        self.op = ""
        self.total = 0

        row, column = 1, 0
        buttons = ["7", "8", "9", "/",
                   "4", "5", "6", "*",
                   "1", "2", "3", "-",
                   "C", "0", ".", "+", "="]
        for button in buttons:
            self.build_button(button, row, column)
            column += 1
            if column > 3:
                column = 0
                row += 1
        self.master.bind("<Key>", self.key_press)

    def key_press(self, event):
        key = event.char
        if key == "\r":            # Enter
            self.operation(); return
        if key == "\x08":          # Backspace → acá limpia todo
            self.clear_display(); return
        if key == "\x1B":          # Escape
            self.master.quit(); return
        if key in "0123456789.+-*/":
            self.click(key)

    def build_button(self, button, row, col):
        if button == "C":
            b = tk.Button(master=self.master, text=button, width=3,
                          command=self.clear_display)   # sin lambda: no hay args
        elif button == "=":
            b = tk.Button(master=self.master, text=button, width=3,
                          command=self.operation)
        else:
            b = tk.Button(master=self.master, text=button, width=3,
                          command=lambda btn=button: self.click(btn))
            # ↑ default-arg capture: sin `btn=button` todos los botones usarían el último
        b.grid(column=col, row=row)

    def clear_display(self):
        self.display.delete(0, tk.END)
        self.op_verification = False
        self.current = ""
        self.total = 0
        self.op = ""

    def click(self, button):
        if self.op_verification:
            self.display.delete(0, tk.END)
            self.op_verification = False
        self.display.insert(tk.END, button)
        if button in "0123456789" or button == ".":
            self.current += button
        else:
            if self.current and not self.op:
                self.total = float(self.current)
            self.current = ""
            self.op = button
            self.op_verification = True

    def operation(self):
        if self.current and self.op:
            divisor = float(self.current)
            if self.op == "/":
                if divisor == 0:
                    self.display.delete(0, tk.END)
                    self.display.insert(tk.END, "Error"); return
                self.total /= divisor
            elif self.op == "+": self.total += divisor
            elif self.op == "*": self.total *= divisor
            elif self.op == "-": self.total -= divisor
        self.display.delete(0, tk.END)
        self.display.insert(tk.END, str(self.total))
        self.op = ""               # ← bug corregido: sin esto, "=" dos veces re-aplica la operación
        self.current = str(self.total)

root = tk.Tk()
root.title("Calculadora")
Calculadora(root)
root.mainloop()
```

Correcciones respecto al código original del curso: (1) la captura de la variable del loop en el `lambda` ahora usa el default-arg idiom; (2) `operation()` limpia `self.op` y fija `current` al total, permitiendo encadenar operaciones sin el bug de repetir la última; (3) división por cero mostrada como "Error" en vez de explotar con `ZeroDivisionError` dentro de un callback (que mataría la app silenciosamente en la consola).

## Proyecto 3 — Chat multiusuario (sockets + threads + GUI)

Combina [[Python - Redes y HTTP]] (sockets TCP + TLS), [[Python - Concurrencia Threading Multiprocessing Asyncio]] (thread por cliente) y Tkinter (hilo receptor + cola). El servidor es headless; el cliente tiene GUI.

### `server.py`

```python
#!/usr/bin/env python3
import socket
import threading
import ssl

def client_thread(client_socket, clients, usernames):
    username = client_socket.recv(1024).decode()
    usernames[client_socket] = username
    print(f"\n[+] El usuario {username} se ha conectado al chat")

    for client in clients:
        if client is not client_socket:
            client.sendall(f"\n El usuario {username} ha entrado al chat\n\n".encode())

    while True:
        try:
            message = client_socket.recv(1024).decode()
            if not message:
                break
            if message == "!usuarios":
                client_socket.sendall(
                    f"\n [+] Usuarios conectados: {','.join(usernames.values())}\n\n".encode())
                continue
            for client in clients:
                if client is not client_socket:
                    client.sendall(f"{message}\n".encode())
        except (ConnectionResetError, OSError):
            break

    client_socket.close()
    clients.remove(client_socket)
    del usernames[client_socket]
    for client in clients:                       # avisar la salida a los demás
        client.sendall(f"\n[!] {username} abandonó el chat\n".encode())

def main():
    host, port = "localhost", 1234
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # evita TIME_WAIT
    server_socket.bind((host, port))

    # 🔧 CORRECCIÓN: ssl.wrap_socket() fue eliminado en Python 3.12 (deprecado en 3.7).
    #    La API actual es SSLContext (verificado: hasattr(ssl, 'wrap_socket') → False en 3.14):
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    context.load_cert_chain(certfile="server-cert.pem", keyfile="server-key.key")
    server_socket = context.wrap_socket(server_socket, server_side=True)

    server_socket.listen()
    print("\n[+] Servidor en escucha...")

    clients, usernames = [], {}
    while True:
        client_socket, address = server_socket.accept()
        clients.append(client_socket)
        print(f"\n[+] Conexión de: {address}")
        thread = threading.Thread(target=client_thread,
                                  args=(client_socket, clients, usernames))
        thread.daemon = True     # se cierran con el proceso; en producción: join+shutdown
        thread.start()

if __name__ == "__main__":
    main()
```

### `client.py`

```python
#!/usr/bin/env python3
import socket
import threading
import ssl
import tkinter as tk
from tkinter.scrolledtext import ScrolledText

def send_message(client_socket, username, text_widget, entry_widget):
    message = entry_widget.get()
    if not message:
        return
    client_socket.sendall(f"{username}> {message}".encode())
    entry_widget.delete(0, tk.END)
    text_widget.configure(state="normal")
    text_widget.insert(tk.END, f"{username} > {message}\n")
    text_widget.configure(state="disable")

def receive_message(client_socket, text_widget):
    """Corre en un hilo daemon: NUNCA toca widgets directamente.
    En este ejemplo simplificado sí inserta desde el hilo (así lo hacía el curso);
    la versión robusta encola y drena con root.after — ver 'Hilos y mainloop'."""
    while True:
        try:
            message = client_socket.recv(1024).decode()
            if not message:
                break
            text_widget.configure(state="normal")
            text_widget.insert(tk.END, message)
            text_widget.configure(state="disable")
            text_widget.see(tk.END)
        except OSError:
            break

def main():
    host, port = "localhost", 1234
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # 🔧 CORRECCIÓN: también aquí, API moderna (wrap_socket del módulo eliminado en 3.12):
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
    context.check_hostname = False      # solo contra localhost con cert autofirmado
    context.verify_mode = ssl.CERT_NONE # ⚠ producción: load_verify_locations + CERT_REQUIRED
    client_socket = context.wrap_socket(client_socket, server_hostname=host)

    client_socket.connect((host, port))
    username = input("\n[+] Introduce tu usuario: ")
    client_socket.sendall(username.encode())

    window = tk.Tk()
    window.title("Chat")

    text_widget = ScrolledText(window, state="disable")
    text_widget.pack(padx=15, pady=15)

    frame_widget = tk.Frame(window)
    frame_widget.pack(fill=tk.BOTH, expand=1, pady=5, padx=5)

    entry_widget = tk.Entry(frame_widget)
    entry_widget.bind("<Return>",
        lambda e: send_message(client_socket, username, text_widget, entry_widget))
    entry_widget.pack(side=tk.LEFT, fill=tk.X, expand=1)

    tk.Button(frame_widget, text="Enviar",
        command=lambda: send_message(client_socket, username, text_widget, entry_widget)
    ).pack(side=tk.RIGHT, padx=5)
    tk.Button(frame_widget, text="Listado de Usuarios",
        command=lambda: client_socket.sendall("!usuarios".encode())).pack(side=tk.RIGHT)
    tk.Button(frame_widget, text="Salir",
        command=lambda: (client_socket.sendall(f"[!] {username} salió\n".encode()),
                         client_socket.close(), window.destroy())).pack(side=tk.RIGHT)

    thread = threading.Thread(target=receive_message,
                              args=(client_socket, text_widget), daemon=True)
    thread.start()

    window.mainloop()
    client_socket.close()

if __name__ == "__main__":
    main()
```

Lecciones del proyecto:

- **`ssl.wrap_socket` ya no existe** (eliminado en 3.12, verificado en 3.14): `SSLContext.wrap_socket()` es la API — y además obliga a decidir explícitamente `verify_mode`/`check_hostname`, que es lo correcto.
- El `recv` en hilo con `ScrolledText(state="disable")` evita que el usuario edite el log; habilitar/deshabilitar alrededor de cada escritura.
- `SO_REUSEADDR` evita `Address already use` al reiniciar el server (TIME_WAIT); ver [[Comandos Básicos de Sistema y Red en Linux]] para diagnóstico con `ss`/`netstat`.

## Errores comunes

1. **`TclError: ... geometry manager pack inside ... grid`** → gestores mezclados en el mismo contenedor; anidar por frames.
2. **Widget invisible** → falta `pack/grid/place`, o quedó fuera del viewport (falta `expand=True`).
3. **Layout no escala al redimensionar** → falta `weight` en `grid`/`pack(expand=True)`.
4. **Actualizar GUI desde un hilo** → crashes intermitentes; patrón cola + `after`.
5. **Imagen fantasma** → referencia local a `PhotoImage` recolectada por GC; guardarla en `self`.
6. **`command=funcion()`** con paréntesis → se ejecuta al construir, no al clickear; sin paréntesis o con `lambda`.
7. **Lambda en loop captura la variable** → `command=lambda i=i: f(i)` (default-arg) — ver Proyecto 2.
8. **`mainloop()` no llamado** → ventana que aparece congelada o no aparece.
9. **`askopenfilename` cancelado sin chequear** → `open("")` → `FileNotFoundError`.
10. **Cerrando con `quit()` en vez de `destroy()`** → el proceso queda vivo sin ventana (zombie).
11. **Mezclar `text=` y `textvariable=`** en el mismo widget → se pisan; elegir uno.

## Cheatsheet

| Elemento | API | Una línea |
|---|---|---|
| Raíz | `root = tk.Tk(); root.mainloop()` | Singleton + bucle de eventos |
| Tamaño | `root.geometry("800x400+50+50")` | `WxH+X+Y` |
| Timer | `root.after(ms, fn)` | El único "reloj" seguro |
| Pack | `.pack(side, fill, expand)` | Bandas |
| Grid | `.grid(row, column, sticky)` + `row/columnconfigure(weight=1)` | Formularios |
| Place | `.place(relx, rely, anchor)` | Overlays |
| Texto | `Label` / `Entry.get()` / `Text.get("1.0", "end-1c")` | get siempre explícito |
| Variables | `StringVar/IntVar/BooleanVar` + `textvariable` | Espejo observable |
| Eventos | `widget.bind("<Key>", fn)` | Handler recibe `event` |
| Virtual | `event_generate("<<X>>")` + `bind("<<X>>", fn)` | Eventos de alto nivel |
| Diálogos | `messagebox.askyesno`, `filedialog.askopenfilename` | Nativos del SO |
| ttk | `ttk.Combobox/Treeview/Notebook/Progressbar` | Widgets themables |
| Temas | `ttk.Style().theme_use("clam")` | y `configure("TButton", ...)` |
| Hilos | `queue.Queue` + `root.after` drenando | Jamás tocar widgets desde hilos |
| Imagen | `ImageTk.PhotoImage` guardado en `self` | Evita el label en blanco |
| Empaquetar | `pyinstaller --onefile --windowed` | Ejecutable sin Python |

## Relacionadas

- [[Python - Fundamentos y Sintaxis]] — clases, lambdas y closures que sostienen los callbacks
- [[Python - Programación Orientada a Objetos]] — estructurar apps GUI con POO (los tres proyectos)
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — el patrón hilo + Queue del chat
- [[Python - Redes y HTTP]] — sockets TCP/TLS del servidor de chat; `http.server` como backend
- [[Python - Manejo de Archivos y Serialización]] — abrir/guardar del editor; UTF-8 explícito
- [[Python - Módulos, Paquetes y Distribución]] — PyInstaller y distribución del ejecutable
- [[Python - Librerías Estándar Esenciales]] — `tkinter`, `queue`, `ssl` en el mapa de la stdlib
