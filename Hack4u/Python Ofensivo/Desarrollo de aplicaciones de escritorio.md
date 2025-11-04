----
# Introducción a las Interfaces (GUI)

**Tkinter** es una biblioteca estándar de Python para la creación de interfaces gráficas de usuario (GUI). Es una interfaz de programación para ‘**Tk**‘, un toolkit de GUI que es parte de Tcl/Tk. Tkinter es notable por su simplicidad y eficiencia, siendo ampliamente utilizado en aplicaciones de escritorio y herramientas educativas. Ademas que para esto los conceptos como la [[Programación Orientada a Objetos]], [[Manejo de Librerías]] y [[Biblioteca Estándar y Herramientas Adicionales]] son muy importantes
## **¿Por qué Tkinter?**

- **Facilidad de Uso**: Tkinter es amigable para principiantes. Su estructura sencilla y clara lo hace ideal para aprender los conceptos básicos de la programación de GUI.
- **Portabilidad**: Las aplicaciones creadas con Tkinter pueden ejecutarse en diversos sistemas operativos sin necesidad de modificar el código.
- **Amplia Disponibilidad**: Al ser parte de la biblioteca estándar de Python, Tkinter está disponible por defecto en la mayoría de las instalaciones de Python, lo que elimina la necesidad de instalaciones adicionales.

En esta sección del curso, nos sumergiremos en el mundo de Tkinter, empezando con una introducción detallada que nos permitirá entender y utilizar sus múltiples funcionalidades. A través de proyectos prácticos, aplicaremos estos conocimientos para construir desde aplicaciones simples hasta interfaces más complejas, proporcionando una base sólida para cualquiera interesado en el desarrollo de GUI con Python.

# Desarrollo de aplicaciones GUI en Tkinter

## **Tkinter: Explorando Componentes Clave**

1. **tk.Label**

- **Descripción**: ‘**tk.Label**‘ es un widget en Tkinter utilizado para mostrar texto o imágenes. El texto puede ser estático o actualizarse dinámicamente.
- **Uso Común**: Se usa para añadir etiquetas informativas en una GUI, como títulos, instrucciones o información de estado.
- **Características Clave**:
    - **text**: Para establecer el texto que se mostrará.
    - **font**: Para personalizar la tipografía.
    - **bg y fg**: Para establecer el color de fondo (bg) y de texto (fg).
    - **image**: Para mostrar una imagen.
    - **wraplength**: Para especificar a qué ancho el texto debería envolverse.

1. **mainloop()**

- **Descripción**: ‘**mainloop()**‘ es una función esencial en Tkinter que ejecuta el bucle de eventos de la aplicación. Este bucle espera eventos, como pulsaciones de teclas o clics del mouse, y los procesa.
- **Importancia**: Sin ‘**mainloop()**‘, la aplicación GUI no responderá a eventos y parecerá congelada. Es lo que mantiene viva la aplicación.

**3. pack()**

- **Descripción**: ‘**pack()**‘ es un método de geometría usado para colocar widgets en una ventana.
- **Funcionalidad**: Organiza los widgets en bloques antes de colocarlos en la ventana. Los widgets se “empaquetan” en el orden en que se llama a ‘**pack()**‘.
- **Características Clave**:
    - **side**: Para especificar el lado de la ventana donde se ubicará el widget (por ejemplo, top, bottom, left, right).
    - **fill**: Para determinar si el widget se expande para llenar el espacio disponible.
    - **expand**: Para permitir que el widget se expanda para ocupar cualquier espacio adicional en la ventana.

**4. grid()**

- **Descripción**: ‘**grid()**‘ es otro método de geometría utilizado en Tkinter para colocar widgets.
- **Funcionalidad**: Organiza los widgets en una cuadrícula. Se especifica la fila y la columna donde debe ir cada widget.
- **Características Clave**:
    - **row y column**: Para especificar la posición del widget en la cuadrícula.
    - **rowspan y columnspan**: Para permitir que un widget ocupe múltiples filas o columnas.
    - **sticky**: Para determinar cómo se alinea el widget dentro de su celda (por ejemplo, N, S, E, W).
### **Conclusión**

Estos componentes de Tkinter (tk.Label, mainloop(), pack(), grid()) son fundamentales para la creación de aplicaciones GUI eficientes y atractivas. Comprender su funcionamiento y saber cómo implementarlos adecuadamente es crucial para cualquier desarrollador que busque crear interfaces de usuario interactivas y funcionales con Python y Tkinter.

```python
#!/usr/bin/env python3

import tkinter as tk

def accion_botton():
    print(f"[+] Se ha presionado el botton")

root = tk.Tk() # Esto es nesesario ya que es una especie de ventana principal

#root.title("Mi primera aplicación") # Este es el titulo de la ventana

label = tk.Label(root, text="!Hola Mundo") # es una especie de widget que nos permite interactuar o pasar información, esto lo crea mas no lo representa
button = tk.Button(root, text="Presióname para que se tense", command=accion_botton)

label.pack() # Nos premite representar la información del label en la ventana
button.pack()

root.mainloop() # Este metodo nos permite ejecutar y manejar todos los eventos que se realizan en al aplicación
```

```python
#!/usr/bin/env python3

import tkinter as tk

root = tk.Tk()
root.title("Metodo Pack()")

label1 = tk.Label(root, text="Mi primer label",bg="red") # EL bg nos permite definir un color de fondo
label2 = tk.Label(root, text="Mi segundo label",bg="green") # EL bg nos permite definir un color de fondo
label3 = tk.Label(root, text="Mi tercer label",bg="blue") # EL bg nos permite definir un color de fondo

label1.pack(fill = tk.X) # tk.X nos permite que los bg tomen todo el ancho y largo de la pantalla
label2.pack(fill = tk.X)
label3.pack(side=tk.LEFT, fill=tk.Y)  # Con ayuda de side podemos posiicónar nuestros label tenemos(LEFT, RIGTH, BOTTON, TOP)

root.mainloop()

```

```python
#!/usr/bin/env python3

import tkinter as tk

root = tk.Tk()
root.title("Metodo Grid()")

label1 = tk.Label(root, text="Mi primer label",bg="red") # EL bg nos permite definir un color de fondo
label2 = tk.Label(root, text="Mi segundo label",bg="green") # EL bg nos permite definir un color de fondo
label3 = tk.Label(root, text="Mi tercer label",bg="blue") # EL bg nos permite definir un color de fondo

label1.grid(row=0, column=0) # Cuando hacemos uso de grid tenemos que especificar en una especiaa tabla execel en la cual se representa
label2.grid(row=0, column=1)
label3.grid(row=1, column=0, columnspan=2)  

root.mainloop()

```

## **Tkinter: Profundizando en Componentes y Gestión de Layout**

1. **place()**
- **Descripción**: ‘**place()**‘ es un método de gestión de geometría en Tkinter que permite posicionar widgets en ubicaciones específicas mediante coordenadas x-y.
- **Características Clave**:
    - **‘x’ y ‘y’**: Especifican la posición del widget en términos de coordenadas.
    - **width y height**: Definen el tamaño del widget.
    - **anchor**: Determina desde qué punto del widget se aplican las coordenadas (por ejemplo, ‘**nw**‘ para esquina superior izquierda).
    - **Posiciones Relativas**: Se pueden utilizar valores relativos (por ejemplo, ‘**relx**‘, ‘**rely**‘) para posicionar widgets en relación con el tamaño de la ventana, lo que hace que la interfaz sea más adaptable al cambiar el tamaño de la ventana.

1. **tk.Entry()**

- **Descripción**: ‘**tk.Entry()**‘ es un widget en Tkinter que permite a los usuarios introducir una línea de texto.
- **Uso Común**: Ideal para campos de entrada de texto como nombres de usuario, contraseñas, etc.
- **Funcionalidades Clave**:
    - **get()**: Para obtener el texto del campo de entrada.
    - **delete()**: Para borrar texto del campo de entrada.
    - **insert()**: Para insertar texto en una posición específica.

**3. tk.Button()**

- **Descripción**: ‘**tk.Button()**‘ es un widget que los usuarios pueden presionar para realizar una acción.
- **Uso Común**: Ejecutar una función o comando cuando se hace clic en él.
- **Características Clave**:
    - **text**: Define el texto que aparece en el botón.
    - **command**: Establece la función que se llamará cuando se haga clic en el botón.

**4. geometry()**

- **Descripción**: ‘**geometry()**‘ es una función que define las dimensiones y la posición inicial de la ventana principal.
- **Funcionalidad**: Permite especificar el tamaño y la ubicación de la ventana en el formato ‘**ancho x alto + X + Y**‘.
- **Importancia**: Es fundamental para establecer el tamaño inicial de la ventana y su posición en la pantalla.

**5. tk.Text()**

- **Descripción**: ‘**tk.Text()**‘ es un widget que permite la entrada y visualización de múltiples líneas de texto.
- **Uso Común**: Ideal para campos de texto más extensos, como áreas de comentarios, editores de texto, etc.
- **Funcionalidades Clave**:
    - Similar a ‘**tk.Entry()**‘, pero diseñado para manejar texto de varias líneas.
    - Permite funciones como copiar, pegar, seleccionar texto.

### **Conclusión**

Estos componentes y funciones (place(), tk.Entry(), tk.Button(), geometry(), tk.Text()) son esenciales en la construcción de interfaces de usuario ricas y funcionales con Tkinter. Proporcionan la flexibilidad necesaria para crear aplicaciones GUI interactivas y atractivas, adaptándose a una amplia gama de necesidades de diseño de interfaz.

```python
#!/usr/bin/env python3

import tkinter as tk

root = tk.Tk()
root.geometry('800x400') # Esto nos permite definir el tamaño de la pantalla con la que se habrira nuestro programa
root.title("Metodo place()")

label1 = tk.Label(root, text="Mi primer label",bg="red") # EL bg nos permite definir un color de fondo
label2 = tk.Label(root, text="Mi segundo label",bg="green") # EL bg nos permite definir un color de fondo
label3 = tk.Label(root, text="Mi tercer label",bg="blue") # EL bg nos permite definir un color de fondo

label1.place(x=20, y=20) # Esto es posicionamiento absoluto, lo que quiere decir que vamos a mover 20px en x y 
label2.place(relx=0.8, rely=0.2) # Esto nos permite movernos de manera relativa osea que dependiendo del tamaño de la pantalla va distanciar la información
label3.place(relx=0.5 , rely=0.5, anchor=tk.CENTER) #Con ayuda de anchor podemos centrar sin imortar el tamño de la ventana

root.mainloop()
```

```python
#!/usr/bin/env python3

import tkinter as tk

root = tk.Tk()
root.geometry('800x400') # Esto nos permite definir el tamaño de la pantalla con la que se habrira nuestro programa
root.title("Metodo place()")

label1 = tk.Label(root, text="Mi primer label",bg="red") # EL bg nos permite definir un color de fondo
label2 = tk.Label(root, text="Mi segundo label",bg="green") # EL bg nos permite definir un color de fondo
label3 = tk.Label(root, text="Mi tercer label",bg="blue") # EL bg nos permite definir un color de fondo

label1.place(x=20, y=20) # Esto es posicionamiento absoluto, lo que quiere decir que vamos a mover 20px en x y 
label2.place(relx=0.8, rely=0.2) # Esto nos permite movernos de manera relativa osea que dependiendo del tamaño de la pantalla va distanciar la información
label3.place(relx=0.5 , rely=0.5, anchor=tk.CENTER) #Con ayuda de anchor podemos centrar sin imortar el tamño de la ventana

root.mainloop()
```

## **Tkinter: Explorando Widgets Avanzados y Funcionalidades de Diálogo**

1. **Frame()**

- **Descripción**: ‘**Frame()**‘ es un widget en Tkinter utilizado como contenedor para otros widgets.
- **Uso Común**: Organizar el layout de la aplicación, agrupando widgets relacionados.
- **Características Clave**:
    - Actúa como un contenedor invisible que puede contener otros widgets.
    - Útil para mejorar la organización y la gestión del layout en aplicaciones complejas.

1. **Canvas()**

- **Descripción**: ‘**Canvas()**‘ es un widget que proporciona un área para dibujar gráficos, líneas, figuras, etc.
- **Funciones de Dibujo**:
    - **create_oval()**: Crea figuras ovales o círculos. Los parámetros especifican las coordenadas del rectángulo delimitador.
    - **create_rectangle()**: Dibuja rectángulos. Los parámetros definen las coordenadas de las esquinas.
    - **create_line()**: Permite dibujar líneas. Se especifican las coordenadas de inicio y fin de la línea.
- **Uso Común**: Crear gráficos, interfaces de juegos, o elementos visuales personalizados.

1. **Menu()**

- **Descripción**: ‘**Menu()**‘ se utiliza para crear menús en una aplicación Tkinter.
- **Uso Común**: Añadir barras de menús con opciones como ‘**Archivo**‘, ‘**Editar**‘, ‘**Ayuda**‘, etc.
- **Características Clave**:
    - Se pueden crear menús desplegables y menús contextuales.
    - Los menús pueden contener comandos, opciones de selección y otros submenús.

1. **messagebox**

- **Descripción**: ‘**messagebox**‘ es un módulo en Tkinter que proporciona ventanas emergentes de diálogo.
- **Funciones Comunes**:
    - **showinfo(), showwarning(), show error()**: Muestran mensajes informativos, de advertencia y de error, respectivamente.
- **Uso Común**: Informar al usuario sobre eventos, confirmaciones, errores o advertencias.

1. **filedialog**

- **Descripción**: ‘**filedialog**‘ es un módulo que ofrece diálogos para seleccionar archivos y directorios.
- **Funciones Clave**:
    - **askopenfilename()**: Abre un cuadro de diálogo para seleccionar un archivo para abrir.
    - **asksaveasfilename()**: Abre un cuadro de diálogo para seleccionar la ubicación y el nombre del archivo para guardar.
    - **askdirectory()**: Permite al usuario seleccionar un directorio.
- **Uso Común**: Integrar la funcionalidad de apertura y guardado de archivos en aplicaciones.

### **Conclusión**

El dominio de estos widgets y módulos (Frame(), Canvas(), Menu(), messagebox, filedialog) es crucial para desarrollar aplicaciones GUI interactivas y completas en Tkinter. Cada uno aporta funcionalidades específicas que permiten crear interfaces de usuario más ricas y dinámicas, adaptadas a una amplia variedad de necesidades.

```python
#!/usr/bin/env python3

import tkinter as tk

root = tk.Tk()
root.title("Frame() Widget")

frame = tk.Frame(root, bg="blue", bd=5) # Frame es una especie de contenedor la cual no hace nada a menos que la usemos para contener algo, aqui definimos sus atributos
frame.place(relx=0.5, rely=0.5, anchor=tk.CENTER) # En este punto definimos en donde queremos que este

label1 = tk.Label(frame, text="Esto es mi label1 de prueba", bg="green") # ya aqui usamos y decimos que nuestro label lo vamos a ubircar dentro de frame
label2 = tk.Label(frame, text="Label2", bg="red") # ya aqui usamos y decimos que nuestro label lo vamos a ubircar dentro de frame

label1.pack(fill=tk.X) # mostramos label
label2.pack(fill=tk.X) # mostramos label

root.mainloop()
```

```python
#!/usr/bin/env python3

import tkinter as tk

root = tk.Tk()
root.title("Canvas() Widget")

canvas = tk.Canvas(root, width=500, height=500, bg="white" ) # el definir un canva no es mas que representar una especie de lienzo en el cual podemos representar diferentes cosas
canvas.pack(pady=20)

oval = canvas.create_oval(50,50, 150,150, fill="red") # Usamo una propiedad la cual no permite crear cirulo, se definien dos  puntos (50,50) y 150,150 y el color
rect = canvas.create_rectangle(200, 50, 350,100, fill="green")
line = canvas.create_line(50,250, 200, 320, fill="blue", width=5)

root.mainloop()
```

```python
#!/usr/bin/env python3

import tkinter as tk
from tkinter import messagebox

root = tk.Tk()
root.title("Menu()" "Widget")

def accion_menu():
    messagebox.showinfo("Menú", "Se ha tensado la cosa") # nos permite lanzar ventanas informativas, recive dos parametros que son el titulo para la ventana y el mensaje

barra_menu = tk.Menu(root)
root.config(menu=barra_menu) # Le decimos que el menu o barra menu principal va a ser barra_menu

menu1 = tk.Menu(barra_menu, tearoff=0) # Creamos ahora un nuevo menu pero que pertenece a barra_menu mas no a root
# con tearoff=0 eliminas las lineas que speravan a nuestras opciones
barra_menu.add_cascade(label="Menú", menu=menu1) # Tenemos que agregar nuestro menu, con label le damos un nombre

menu1.add_command(label="Opcion1") # En este punto creamos opciones las cuales vamos a ver al precionar en menu
menu1.add_command(label="Opcion2")

menu2 = tk.Menu(barra_menu, tearoff=0) # Creamos un nuevo menu
barra_menu.add_cascade(label="Menú 2", menu=menu2)

menu2.add_command(label="Opción 1") # Agregamos opiciones al menu
menu2.add_command(label="Mensaje", command=accion_menu)

root.mainloop()
```

```python
#!/usr/bin/env python3

import tkinter as tk
from tkinter import messagebox

    

root = tk.Tk()

def enviar_mensaje():
    messagebox.showinfo("Mensaje", "Has enviado el mensaje")

botton = tk.Button(text="Enviar", command=enviar_mensaje)
botton.pack()

root.mainloop()
```

```python
#!/usr/bin/env python3

import tkinter as tk
from tkinter import messagebox

root = tk.Tk()

def enviar_mensaje():
    messagebox.showinfo("Mensaje", "Has enviado el mensaje")

botton = tk.Button(text="Enviar", command=enviar_mensaje)
botton.pack()

root.mainloop()
```

## Proyecto 1 (Blog de Notas)

En esta clase, abordaremos un proyecto que consolidará todo lo aprendido hasta ahora en Tkinter: la creación de nuestro propio editor de texto, similar al Bloc de Notas.

Este será un excelente ejercicio para aplicar nuestras habilidades en un contexto práctico y realista, permitiéndonos ver cómo los componentes individuales de Tkinter se unen para formar una aplicación funcional.

Nuestro editor de texto incluirá funcionalidades básicas como:

- Crear un nuevo archivo
- Abrir y editar archivos existentes
- Guardar los cambios
- Cerrar la aplicación

Para esto, haremos uso de varios widgets y técnicas de Tkinter que hemos estudiado, como ‘**tk.Text**‘ para el área de edición, ‘**Menu()**‘ para la barra de menús y ‘**filedialog**‘ para la gestión de archivos.

Adoptaremos un enfoque de Programación Orientada a Objetos (POO) en este proyecto. Utilizaremos clases para estructurar nuestro código, lo que nos permitirá dividir la funcionalidad de la aplicación en bloques lógicos y reutilizables. Este método no solo facilitará la organización y gestión del código, sino que también nos proporcionará una sólida base para el mantenimiento y la escalabilidad futura del proyecto.

Este proyecto no solo nos permitirá practicar la integración de los diferentes componentes y módulos de Tkinter, sino que también nos dará la oportunidad de experimentar con el diseño de interfaces de usuario y el manejo de archivos de texto.

En esta clase, continuaremos con nuestro proyecto del editor de texto, con el objetivo de finalizarlo. Hasta ahora, hemos logrado implementar funciones clave como la creación, edición, guardado y apertura de archivos de texto. En esta sesión, nos enfocaremos en pulir nuestra aplicación, asegurándonos de que todas las funcionalidades trabajen de manera fluida y eficiente.

```python
#!/usr/bin/env python3

import tkinter as tk
from tkinter import filedialog, messagebox

class SimpleTextEditor:
    def __init__(self, root):
        self.root = root
        self.text_area = tk.Text(self.root, )
        self.text_area.pack(fill=tk.BOTH, expand=1) # Con tk.BOTH y expand=1 podemos tener todo el espacio de la ventan
        self.current_open_file = ''
    
    def quit_confirm(self):
        if messagebox.askokcancel("Salir", "Estas seguro que deseas Salir?"):
            self.root.destroy() # Con ayuda de la función destroy podemos eliminar o cerrear o destruir la ventana principal

    def open_file(self):
        filename = filedialog.askopenfilename() # Nos devulve la ruta obsoluta del archivo que escojamos

        if filename:

            if self.text_area.get("1.0", tk.END):
                opt = messagebox.askyesno("Eliminar", "Tienes información sin Guardar, Deseas eliminarla para abrir el archivo?") # mensaje que devuelve true o false
                if opt:
                    self.text_area.delete("1.0", tk.END) # Esta es una forma con la cual podemos eliminar el texto que tengamos en el area antes de abrir un archivo
                    with open(filename, "r") as f:
                        self.text_area.insert("1.0", f.read())

            self.current_open_file = filename

    def new_file(self):
        if self.text_area.get("1.0", tk.END):
            opt = messagebox.askyesno("Eliminar", "Tienes información sin Guardar, Deseas crerar un nuevo archivo?")
            if opt:
                self.text_area.delete("1.0", tk.END) # Esta es una forma con la cual podemos eliminar el texto que tengamos en el area antes de abrir un archivo
                self.current_open_file=''

    def save_file(self):
        if not self.current_open_file:
            filename = filedialog.asksaveasfilename() # Nos permite indicar la ruta y nombre del archivo a guardar
            
            if filename:
                self.current_open_file = filename
            else:
                return
        
        with open(self.current_open_file, "w") as f:
            f.write(self.text_area.get("1.0", tk.END))

root = tk.Tk()

editor = SimpleTextEditor(root)

menu_bar = tk.Menu(root) # Creamos nuestro menu principal
menu_options = tk.Menu(menu_bar, tearoff=0) # Creamos un submenu para las opciones

# Definimos las opciones para el menu
menu_options.add_command(label="Nuevo", command=editor.new_file) 
menu_options.add_command(label="Abrir", command=editor.open_file)
menu_options.add_command(label="Guardar", command=editor.save_file)
menu_options.add_command(label="Salir", command=editor.quit_confirm)

root.config(menu=menu_bar) # Configuramos que menu_bar va a ser el menu principal
menu_bar.add_cascade(label="Archivo", menu=menu_options)

root.mainloop()
```

## Proyecto 2

Una vez que hayamos completado y perfeccionado nuestro editor de texto, daremos un paso adelante en nuestro aprendizaje con el inicio de un nuevo y fascinante proyecto: la creación de una calculadora con interfaz gráfica interactiva. Este proyecto nos permitirá aplicar y expandir aún más nuestros conocimientos en Tkinter, abordando nuevos desafíos y explorando diferentes aspectos de la creación de GUIs.

En la calculadora, implementaremos funciones básicas como:

- Sumar
- Restar
- Multiplicar
- Dividir

Así como una interfaz de usuario intuitiva que permita a los usuarios interactuar fácilmente con la aplicación. Este proyecto no solo reforzará nuestras habilidades en Tkinter y la programación orientada a objetos, sino que también nos brindará la oportunidad de abordar problemas de lógica de programación y diseño de interfaces de usuario.

Con estos dos proyectos, nuestro editor de texto y la calculadora, estaremos no solo consolidando nuestros conocimientos teóricos, sino también ganando experiencia práctica valiosa en el desarrollo de aplicaciones de escritorio con Python y Tkinter. Estos proyectos nos servirán como excelentes ejemplos de lo que podemos lograr y serán una base sólida para futuros proyectos más complejos.

```python
#!/usr/bin/env python3

import tkinter as tk

class Calculadora:

    def __init__(self, master): # A master la vamos a usar como al ventana principal cuando sea nesesario
        self.master = master
        self.display = tk.Entry(self.master, width=15, font=("Arial", 23), bd=10, insertwidth=1, bg="#6495DE", justify="right" ) # Generamos un campo de entrada
        self.display.grid(row=0, column=0, columnspan=4, padx=10, pady=10) # colocamos el campo de entrada
        self.op_verification = False
        self.current = ""
        self.op = ""
        self.total = 0

        row = 1
        column = 0

        buttons = [
            "7", "8", "9", "/",
            "4", "5", "6", "*",
            "1", "2", "3", "-",
            "C", "0", ".", "+", "="
        ]

        for button in buttons: 
            self.build_button(button, row, column)
            column += 1

            if column > 3:
                column=0
                row += 1

        self.master.bind("<Key>", self.key_press) # metodo especial que nos permite detectar las pulsaciones del teclado

    def key_press(self, event):
        key = event.char

        if key == "\\r":
            self.operation()
            return
        elif key == "\\x08":
            self.clear_display()
            return
        elif key == "\\x1B":
            self.master.quit()
            return

        self.click(key)

    def build_button(self,button, row, col):
        if button == "C":
            b = tk.Button(self.master, text=button, width=3, command=lambda: self.clear_display()) # a través de las lambda función podemos evitar que se ejute la función sin que sea llamada
        elif button == "=":
            b = tk.Button(self.master, text=button, width=3, command=lambda: self.operation())
        else:
            b = tk.Button(self.master, text=button, width=3, command=lambda: self.click(button))
        b.grid(column=col, row=row)

    def clear_display(self):
        self.display.delete(0, tk.END)
        self.op_verification = False
        self.current=""
        self.total=0
        self.op=""

    def click(self, button): # Nos permite identificar los clicks dentro del GUI

        if self.op_verification:
            self.op_verification = False

        self.display.insert(tk.END, button)

        if button in "0123456789" or button == ".":

            self.current += button
        else:
            if self.current:
                if not self.op:
                    self.total = float(self.current)

            self.current=""

            self.op = button
            self.op_verification = True
            self.op = button

    def operation(self): # Nos permite identrificar las diferentes operaciones y ejecutarlas

        self.current = self.current
        if self.current and self.op:
            if self.op == "/":
                self.total /= float(self.current)

            if self.op == "+":
                self.total += float(self.current)

            if self.op == "*":
                self.total *= float(self.current)

            if self.op == "-":
                self.total -= float(self.current)

        self.display.delete(0, tk.END)
        self.display.insert(tk.END, str(self.total))

root = tk.Tk() # Ventana principal
my_gui = Calculadora(root)

root.mainloop()
```

## Proyecto 3

En esta clase, nos enfocaremos en construir un chat multiusuario desde cero, utilizando conceptos avanzados como threading y socket para gestionar la comunicación en tiempo real. Se empleará Tkinter para crear una interfaz gráfica intuitiva, permitiendo a varios usuarios conectarse y conversar de manera efectiva.

A lo largo de la clase, aprenderemos cómo estas herramientas pueden ser integradas para desarrollar una aplicación de chat robusta y funcional, explorando tanto la programación de back-end como de front-end para una experiencia de usuario completa.

Server.py:

```python
#!/usr/bin/evn python3

import socket
import threading
import ssl

def client_thread(client_socket, clients, usernames):

    username = client_socket.recv(1024).decode() # recivimos el usuario por parte del cliente y lo decodificamos
    usernames[client_socket] = username  # Tenemos el dicionario vasio llamamdo usernames, almacenaremos en este, como llave el client_socket y como valor el nombre de usuario que recivimos

    print(f"\\n[+] El usurio {username} se a conectando al chat")

    for client in clients: # Iteramos atraves de un bucle por todos los clientes almacenaos
        if client is not client_socket: # Con este condicional comprovamos si el cliente no se encuenra en nuestra lista, no ayuda para enviar este mensaje solo a usuario entrantes y no siempre
            client.sendall(f"\\n El usuario {username} a entrado al chat\\n\\n".encode()) # Enviamos el mensaje al cliente

    while True:
        try:
            message = client_socket.recv(1024).decode() # Vamos a estar reciviendo la información
            if not message:
                break

            if message == "!usuarios": # esto es util para listar los usurios conectados
                client_socket.sendall(f"\\n [+] Listado de usuario disponible: {','.join(usernames.values())}\\n\\n".encode()) # con esto enviamos al cliente que lo solicito la lista de usuarios, es por eso que no enviamos en un bucle para todos
                continue # Evitamos que el programa entre al for que tenemos debajo, con esto solo queda en un modo el cual espera la siguienet petición
            
            for client in clients: # iteramos por la lista
                if client is not client_socket:  # Verificamos que no seamos nosotros mismos
                    client.sendall(f"{message}\\n".encode()) # Enviamos el mensaje a los clientes de la conexión de un usuario

        except:
            break

    # Las siguientes lineas se ejecutan cuando se intenta enviar el mensaje a un usuario no conectado
    client_socket.close() # Cierra el socket
    clients.remove(client_socket) # remueve el socket de la lista de clientes
    del usernames[client_socket] # remueve el usuario del dicciónario de usuarios

def main():

    host = "localhost"
    port = 1234

    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Definimos que tipode conexión queremos entablar
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1) # TIME_WAIT
    server_socket.bind((host, port)) # Con bind configuramos el socket para generar la concexion.
    server_socket = ssl.wrap_socket(server_socket, keyfile="server-key.key", certfile="server-cert.pem", server_side=True) # cifrado ssl
    server_socket.listen() # Nos pones en escucha

    print(f"\\n[+]El Servidor se encuentra en eschucha de conexiones entrantes....")

    clients = []
    usernames = {}

    while True:

        client_socket, addres = server_socket.accept() # Aceptamos las conexiónes y almacenamos el socket del cliente al igual que el addres que contiene(ip, puerto)
        clients.append(client_socket) # almacenamos el socket del cliente en nuestra lista de clientes

        print(f"\\n[+] Se a conectando: {addres}")

        thread = threading.Thread(target=client_thread, args=(client_socket, clients, usernames)) # Creamos un hilo y llamamos a la funcion client_thread con las variables que le pasamos a args
        thread.daemon = True # es una forma de tratar de garantizar que cuando se cierre el programa nuestros thread se cierren y no queden en segundo plano
        thread.start() # Iniciamos el hilo creado anterior mente

    server_socket.close()

if __name__ == "__main__":
    main()

```

Cliente.py:

```python
#!/usr/bin/env python3

import socket
import threading
from tkinter import *
import ssl
import tkinter
from tkinter.scrolledtext import ScrolledText

def send_message(client_socket, username, text_widget, entry_widget):

    message = entry_widget.get() # Almacenamos los valores de la entrada de datos
    client_socket.sendall(f"{username}> {message}".encode()) # enviamos el mensaje al nuestro servidor

    entry_widget.delete(0, END) # Limpiamos la entrada de datos para poder seguire imprimiendo
    text_widget.configure(state='normal') # Habilitamos la escriura momentaneamente para representar el mensaje
    text_widget.insert(END, f"{username} > {message}\\n") # incertamos el mensaje en la pantall de texto
    text_widget.configure(state='disable') # Desabilitamos la escritura para que no se inserten mensajes no tramitados

def recive_message(client_socket, text_widget):
    while True:
        try:
            message = client_socket.recv(1024).decode() # Usando el  bucle infinito  vamos a estar reciviendo información siempre.

            if not message:
                break

            text_widget.configure(state='normal') # Habilitamos la escriura momentaneamente para representar el mensaje
            text_widget.insert(END, message) # incertamos el mensaje en la pantall de texto
            text_widget.configure(state='disable') # Desabilitamos la escritura para que no se inserten mensajes no tramitados

        except:
            break

    

def list_users_requests(client_socket):
    client_socket.sendall("!usuarios".encode()) # Envia una petición para que se regrese la lista de usuario conectados

def exit_request(client_socket, username, window):
    client_socket.sendall(f"[!] El usuario {username} ha abandonado el chat \\n\\n".encode()) # Enviamos un mensaje a todos los usuario de que cierto usuario abandono el chat
    client_socket.close() # cerramos el socket del usuario que dio en el boton salir

    window.quit() # Cerramos la ventana de ese usuario
    window.destroy() # Destruimos la ventana del usuario que dio en el boton salir

def main():

    host = "localhost"
    port = 1234

    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Definimos el socket y su tipo que es TCP en este caso
    client_socket = ssl.wrap_socket(client_socket) # cifrado ssl
    client_socket.connect((host, port)) # generamos la coneción con ayuda de connect a un servidor, es por eso que le pasamos una tupla con el host y el puerto

    username = input(f"\\n[+] Introduce tu usuario: ")

    client_socket.sendall(username.encode()) # Enviamos el nombre de usuario a nuestro servidor para que nos reconosca

    window = Tk() # Con esto inicializamos nuestra ventana princiapl
    window.title("Chat") # Le damos un nombre a la ventana principal

    text_widget = ScrolledText(window, state='disable') # Asociamos el widget a la ventana principal y con state desactivamos la escritura en el mismo
    text_widget.pack(padx=15, pady=15) # agregamos visualmente a la pantalla principal

    frame_widget = Frame(window) # Creamos un frame y lo asociamos a nuestra window
    frame_widget.pack(fill=BOTH, expand=1, pady=5, padx=5) # Definimos la reprensentanción del frame con pack

    entry_widget = Entry(frame_widget) # Definimos el entry como parte del frame_widget
    entry_widget.bind("<Return>", lambda _: send_message(client_socket, username, text_widget, entry_widget)) # con ayuda de bind detectamos pulsaciones y llamamos una función lambda pra que no se ejecute de manera automatica
    entry_widget.pack(side=LEFT, fill=X, expand=1) # Representamos el entry en nuestra venana principal

    button_widget = Button(frame_widget, text="Enviar", command=lambda: send_message(client_socket, username, text_widget, entry_widget)) # generamos un botton  lo asociamos a nuestro frame
    button_widget.pack(side=RIGHT, padx=5, pady=5) # con ayuda de pack representamos nuestro frame en la ventana

    user_widget= Button(frame_widget, text="Listado de Usuarios", command=lambda: list_users_requests(client_socket))
    user_widget.pack(fill=BOTH,padx=5, pady=5) # con ayuda de pack representamos nuestro frame en la ventana

    exit_widget= Button(frame_widget, text="Salir", command=lambda: exit_request(client_socket, username, window))
    exit_widget.pack(padx=5, pady=5) # con ayuda de pack representamos nuestro frame en la ventana

    thread = threading.Thread(target=recive_message, args=(client_socket, text_widget)) # Creamos un nuevo hilo y llamamos a una función y le especificamos los argumentos que vamos a pasarle
    thread.daemon = True # Es una garantia para que los hilos no queden en espera o segundo plano
    thread.start() # Iniciamos nuestro hilo

    window.mainloop()

    client_socket.close() # cerramos el socket del cliente

if __name__ == "__main__":
    main()
```