# Estructura de Tags para Desarrollo Web y Programación

> **Propósito**: Taxonomía de etiquetas para reorganizar y consultar las 95 notas actuales.
> **Uso**: Añadir frontmatter YAML a cada nota: `tags: [tag1, tag2, tag3]`

---

## 📂 Estructura Actual (11 subcarpetas, 95 notas)

| Subcarpeta | Notas | Prefijo sugerido |
|------------|-------|------------------|
| **Css** | 24 | `css/` |
| **Java Script** | 25 | `js/` |
| **Python** | 8 | `py/` |
| **GitHub** | 7 | `git/` |
| **Html** | 8 | `html/` |
| **Gulp** | 5 | `gulp/` |
| **Bash Scripting** | 2 | `bash/` |
| **ReactJS** | 3 | `react/` |
| **Scss** | 4 | `scss/` |
| **TailwindCSS** | 2 | `tailwind/` |
| **Docker** | 1 | `docker/` |

---

## 🏷️ Taxonomía de Tags (Jerárquica)

### 1. **Lenguajes / Tecnologías Base** (`lang:`)

```
lang:html          # HTML5, semántica, estructura
lang:css           # CSS3, propiedades, layout
lang:javascript    # ECMAScript, vanilla JS
lang:typescript    # (futuro)
lang:python        # Python 3.x
lang:bash          # Shell scripting
lang:scss          # Sass/SCSS
lang:sql           # (futuro)
```

### 2. **Frameworks / Librerías** (`fw:`)

```
fw:react           # React, hooks, componentes
fw:vue             # (futuro)
fw:angular         # (futuro)
fw:svelte          # (futuro)
fw:tailwind        # Tailwind CSS
fw:bootstrap       # (futuro)
fw:gsap            # GreenSock Animation Platform
fw:parallax        # Parallax.js
fw:modernizr       # Feature detection
fw:gulp            # Task runner
```

### 3. **Herramientas / Entorno** (`tool:`)

```
tool:git           # Git, GitHub, workflows
tool:docker        # Contenedores, compose
tool:vscode        # (futuro)
tool:vim           # (futuro)
tool:cli           # Línea de comandos general
tool:build         # Bundlers, transpilers (vite, webpack)
tool:package       # npm, pip, gestores paquetes
```

### 4. **Conceptos Fundamentales** (`concept:`)

```
concept:syntax     # Sintaxis, gramática, reglas
concept:fundamentals  # Bases, introducción, qué es
concept:oop        # Programación orientada a objetos
concept:fp         # Programación funcional
concept:dom        # Document Object Model
concept:events     # Eventos, listeners, delegación
concept:async      # Promesas, async/await, callbacks
concept:modules    # Import/export, ESM, CommonJS
concept:types      # Tipos de datos, coerción
concept:operators  # Operadores, precedencia
concept:control-flow  # if, switch, loops, try/catch
concept:functions  # Declaración, arrow, closure, scope
concept:data-structures  # Arrays, objetos, Map, Set
concept:patterns   # Patrones de diseño, arquitectura
concept:performance  # Optimización, métricas
concept:security   # XSS, CSRF, sanitización
concept:accessibility  # a11y, ARIA, semántica
concept:responsive # Mobile-first, breakpoints, fluid
concept:animation  # Transiciones, keyframes, GSAP
concept:layout     # Flexbox, Grid, positioning, display
concept:specificity  # Cascada, herencia, especificidad
```

### 5. **Nivel / Progresión** (`level:`)

```
level:basico       # Introducción, fundamentos, "qué es"
level:intermedio   # Uso práctico, patrones comunes
level:avanzado     # Internals, optimización, edge cases
level:experto      # Internals profundos, contribución, RFCs
```

### 6. **Tipo de Contenido** (`type:`)

```
type:reference     # Cheatsheet, referencia rápida, sintaxis
type:tutorial      # Paso a paso, guía guiada
type:concept       # Explicación teórica, mental model
type:pattern       # Patrón reutilizable, snippet
type:troubleshooting  # Errores comunes, debugging, fixes
type:best-practice # Recomendaciones, style guides
type:tooling       # Configuración, setup, workflow
type:project       # Proyecto completo, ejemplo real
type:comparison    # X vs Y, cuándo usar cada uno
```

### 7. **Área de Aplicación** (`area:`)

```
area:frontend      # Navegador, UI, cliente
area:backend       # Servidor, API, base de datos
area:fullstack     # End-to-end
area:devops        # CI/CD, deploy, infraestructura
area:scripting     # Automatización, CLI, tareas
area:testing       # Unit, integration, e2e
area:debugging     # Herramientas, técnicas, profiling
```

### 8. **Estado / Mantenimiento** (`status:`)

```
status:draft       # Borrador, incompleto
status:active      # Actualizado, en uso
status:deprecated  # Obsoleto, legado
status:review      # Pendiente revisión/actualización
status:reference   # Consolidado, fuente de verdad
```

---

## 🎯 Matriz de Aplicación por Subcarpeta

### **Css/** (24 notas) → Tags base: `lang:css`

| Archivo | Tags sugeridos |
|---------|----------------|
| Anatomía de CSS.md | `lang:css, concept:syntax, level:basico, type:reference` |
| CSS Introducción.md | `lang:css, concept:fundamentals, level:basico, type:concept` |
| Box Model.md | `lang:css, concept:layout, level:basico, type:concept` |
| Tipos de Display.md | `lang:css, concept:layout, level:basico, type:reference` |
| Tipos de Posiciones.md | `lang:css, concept:layout, level:intermedio, type:concept` |
| Flexbox.md | `lang:css, concept:layout, level:intermedio, type:concept` |
| CSS Grid.md | `lang:css, concept:layout, level:intermedio, type:concept` |
| Cuando usar GRID o Flexbox.md | `lang:css, concept:layout, level:intermedio, type:comparison` |
| Responsive Web Design.md | `lang:css, concept:responsive, area:frontend, level:intermedio, type:concept` |
| Medidas CSS.md / Medidas Relativas.md | `lang:css, concept:syntax, level:basico, type:reference` |
| Colores CSS.md | `lang:css, concept:syntax, level:basico, type:reference` |
| Background.md | `lang:css, concept:syntax, level:basico, type:reference` |
| Sombras CSS.md | `lang:css, concept:syntax, level:basico, type:reference` |
| Degradados en CSS.md | `lang:css, concept:syntax, level:intermedio, type:reference` |
| Animaciones.md | `lang:css, concept:animation, level:intermedio, type:concept` |
| Efectos de Scroll.md | `lang:css, concept:animation, level:avanzado, type:pattern` |
| Selectores en CSS.md | `lang:css, concept:syntax, level:basico, type:reference` |
| Especificidad.md | `lang:css, concept:specificity, level:intermedio, type:concept` |
| Pseudo Código.md | `lang:css, concept:syntax, level:intermedio, type:reference` |
| Normalize.md | `lang:css, tool:build, level:basico, type:tooling` |
| Fuentes Externas.md | `lang:css, concept:syntax, level:basico, type:tooling` |
| Imágenes CSS.md | `lang:css, concept:syntax, level:basico, type:reference` |
| Material Icons.md | `lang:css, fw:icon, level:basico, type:tooling` |
| Crear archivo para CSS.md | `lang:css, tool:build, level:basico, type:tooling` |
| Estilos para escribir CSS.md | `lang:css, type:best-practice, level:basico` |

---

### **Java Script/** (25 notas) → Tags base: `lang:javascript`

| Archivo | Tags sugeridos |
|---------|----------------|
| Java Script.md | `lang:javascript, concept:fundamentals, level:basico, type:concept` |
| Sintaxis Antigua.md | `lang:javascript, concept:syntax, level:basico, type:reference` |
| Sintaxis Moderna.md | `lang:javascript, concept:syntax, level:intermedio, type:reference` |
| Variables (implícito) | `lang:javascript, concept:types, level:basico` |
| Tipos: Strings, Números, Booleans, Arrays, Objetos | `lang:javascript, concept:types, concept:data-structures, level:basico, type:reference` |
| Operadores.md | `lang:javascript, concept:operators, level:basico, type:reference` |
| Estructuras de Control.md | `lang:javascript, concept:control-flow, level:basico, type:reference` |
| Funciones.md | `lang:javascript, concept:functions, level:intermedio, type:concept` |
| Array Methods.md | `lang:javascript, concept:data-structures, level:intermedio, type:reference` |
| Iteradores.md | `lang:javascript, concept:data-structures, level:intermedio, type:concept` |
| Document Object Model (DOM).md | `lang:javascript, concept:dom, area:frontend, level:intermedio, type:concept` |
| Acceder a Html.md | `lang:javascript, concept:dom, level:basico, type:tutorial` |
| Eventos.md | `lang:javascript, concept:events, area:frontend, level:intermedio, type:concept` |
| Formato Json.md | `lang:javascript, concept:data-structures, level:basico, type:reference` |
| Async: Qué es una API.md | `lang:javascript, concept:async, area:frontend, level:intermedio, type:concept` |
| La Consola.md | `lang:javascript, tool:cli, area:debugging, level:basico, type:tooling` |
| Archivos SVG.md | `lang:javascript, area:frontend, level:intermedio, type:pattern` |
| Librería GSAP.md | `lang:javascript, fw:gsap, concept:animation, level:avanzado, type:tutorial` |
| Librería Parallax.md | `lang:javascript, fw:parallax, concept:animation, level:avanzado, type:tutorial` |
| Recomendaciones.md | `lang:javascript, type:best-practice, level:intermedio` |
| Uso de Java Script.md | `lang:javascript, area:frontend, level:basico, type:concept` |

---

### **Python/** (8 notas) → Tags base: `lang:python`

| Archivo | Tags sugeridos |
|---------|----------------|
| Fundamentos de Python.md | `lang:python, concept:fundamentals, level:basico, type:concept` |
| El Intérprete de Python.md | `lang:python, concept:fundamentals, level:basico, type:concept` |
| Estructuras de Datos en Python.md | `lang:python, concept:data-structures, level:intermedio, type:reference` |
| Clases y Objetos.md | `lang:python, concept:oop, level:intermedio, type:concept` |
| Proyectos de Programación Orientada a Objetos en Python.md | `lang:python, concept:oop, level:avanzado, type:project` |
| Organización, Importación y Distribución de Módulos y Paquetes en Python.md | `lang:python, concept:modules, level:intermedio, type:concept` |
| Manejo de Bibliotecas en Python.md | `lang:python, tool:package, level:intermedio, type:tooling` |
| Librerías Python para Scripting y Redes.md | `lang:python, area:scripting, area:backend, level:avanzado, type:reference` |
| Python Interacción y Archivos.md | `lang:python, area:scripting, level:intermedio, type:tutorial` |
| Introducción a las Interfaces GUI.md | `lang:python, area:frontend, level:intermedio, type:concept` |

---

### **Html/** (8 notas) → Tags base: `lang:html`

| Archivo | Tags sugeridos |
|---------|----------------|
| Html.md | `lang:html, concept:fundamentals, level:basico, type:concept` |
| Estructurar Html.md | `lang:html, concept:syntax, level:basico, type:concept` |
| Etiquetas.md | `lang:html, concept:syntax, level:basico, type:reference` |
| Etiquetas Meta.md | `lang:html, concept:syntax, area:frontend, level:basico, type:reference` |
| Formularios.md | `lang:html, concept:forms, area:frontend, level:intermedio, type:concept` |
| Formatos Avif y Webp.md | `lang:html, concept:performance, area:frontend, level:avanzado, type:best-practice` |
| Lazy Loading.md | `lang:html, concept:performance, area:frontend, level:intermedio, type:pattern` |
| Prefetch.md / Preload.md / Modernizr.md | `lang:html, concept:performance, tool:build, level:avanzado, type:tooling` |

---

### **GitHub/** (7 notas) → Tags base: `tool:git`

| Archivo | Tags sugeridos |
|---------|----------------|
| Qué es Git.md | `tool:git, concept:fundamentals, level:basico, type:concept` |
| Configuración Inicial.md | `tool:git, tool:cli, level:basico, type:tooling` |
| Comandos Git.md | `tool:git, tool:cli, level:basico, type:reference` |
| Ramas.md | `tool:git, concept:patterns, level:intermedio, type:concept` |
| Rebase.md | `tool:git, concept:patterns, level:avanzado, type:concept` |
| Stash.md | `tool:git, tool:cli, level:intermedio, type:reference` |
| Tags.md | `tool:git, concept:patterns, level:intermedio, type:reference` |
| Archivos Especiales.md | `tool:git, tool:build, level:intermedio, type:tooling` |

---

### **ReactJS/** (3 notas) → Tags base: `fw:react`

| Archivo | Tags sugeridos |
|---------|----------------|
| Esenciales.md | `fw:react, concept:fundamentals, level:basico, type:concept` |
| Esenciales2.md | `fw:react, concept:fundamentals, level:intermedio, type:concept` |
| Vite.md | `fw:react, tool:build, level:intermedio, type:tooling` |

---

### **Scss/** (4 notas) → Tags base: `lang:scss`

| Archivo | Tags sugeridos |
|---------|----------------|
| Qué es Sass.md | `lang:scss, concept:fundamentals, level:basico, type:concept` |
| Sintaxis de Sass.md | `lang:scss, concept:syntax, level:basico, type:reference` |
| Separar Sass en Archivos.md | `lang:scss, concept:modules, level:intermedio, type:best-practice` |
| Mixins.md | `lang:scss, concept:patterns, level:intermedio, type:pattern` |

---

### **TailwindCSS/** (2 notas) → Tags base: `fw:tailwind`

| Archivo | Tags sugeridos |
|---------|----------------|
| Instalación de Tailwindcss.md | `fw:tailwind, tool:build, level:basico, type:tooling` |
| Lo Básico.md | `fw:tailwind, concept:fundamentals, level:basico, type:concept` |

---

### **Gulp/** (5 notas) → Tags base: `fw:gulp, tool:build`

| Archivo | Tags sugeridos |
|---------|----------------|
| Gulp.md | `fw:gulp, tool:build, concept:fundamentals, level:basico, type:concept` |
| Instalar Gulp.md | `fw:gulp, tool:build, level:basico, type:tooling` |
| GulpFile Completo.md | `fw:gulp, tool:build, level:intermedio, type:pattern` |
| Compilar Sass.md | `fw:gulp, lang:scss, tool:build, level:intermedio, type:tutorial` |
| Performance Web.md | `fw:gulp, concept:performance, level:avanzado, type:best-practice` |

---

### **Bash Scripting/** (2 notas) → Tags base: `lang:bash, area:scripting`

| Archivo | Tags sugeridos |
|---------|----------------|
| Bash Scripting Fundamentos.md | `lang:bash, concept:fundamentals, level:basico, type:concept` |
| Redirección de ES y Procesos en Bash.md | `lang:bash, concept:advanced, area:scripting, level:intermedio, type:concept` |

---

### **Docker/** (1 nota) → Tags base: `tool:docker, area:devops`

| Archivo | Tags sugeridos |
|---------|----------------|
| Gestión de Contenedores con Docker y Docker Compose.md | `tool:docker, area:devops, level:intermedio, type:tutorial` |

---

## 🔍 Consultas Útiles (Ejemplos)

```bash
# Todas las notas de nivel básico para repasar fundamentos
tags: level:basico

# Referencias rápidas de CSS para consultar sintaxis
tags: lang:css, type:reference

# Patrones de layout modernos (Flexbox + Grid + Tailwind)
tags: concept:layout, level:intermedio

# Todo JavaScript relacionado con DOM y eventos
tags: lang:javascript, concept:dom
tags: lang:javascript, concept:events

# Animaciones (CSS + GSAP + Parallax)
tags: concept:animation

# Herramientas de build y performance
tags: tool:build, concept:performance

# Git workflows intermedios/avanzados
tags: tool:git, level:intermedio
tags: tool:git, level:avanzado

# Python para scripting/automatización
tags: lang:python, area:scripting

# Notas que necesitan revisión
tags: status:review

# Comparativas (cuándo usar qué)
tags: type:comparison
```

---

## 📝 Formato Frontmatter Sugerido

```yaml
---
title: "Nombre de la Nota"
tags:
  - lang:javascript
  - concept:async
  - level:intermedio
  - type:concept
  - area:frontend
created: 2024-01-15
updated: 2024-09-25
status: active
---
```

---

## 🚀 Próximos Pasos

1. **Aplicar tags** a las 95 notas existentes (script automatizable)
2. **Crear índices** por tag (ej. `index_lang_javascript.md`)
3. **Migrar a estructura plana** con tags en lugar de carpetas rígidas
4. **Añadir tags faltantes** conforme crezcan notas (TypeScript, Node.js, Testing, etc.)

---

*Generado automáticamente tras análisis de estructura actual*