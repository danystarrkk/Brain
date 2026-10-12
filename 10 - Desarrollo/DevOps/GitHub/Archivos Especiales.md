---
tags:
  - desarrollo
  - devops
  - archivos-especiales
tipo: concepto
estado: terminado
---

# Archivos Especiales

Git vive rodeado de archivos que controlan su comportamiento: reglas de ignorado (`.gitignore`), atributos de línea/binario (`.gitattributes`), submódulos (`.gitmodules`), hooks (`.git/hooks/`), firmas de identidad (`.mailmap`) y más. Este es el catálogo de referencia de cada uno, con la sintaxis completa y las trampas que producen los bugs más persistentes de Git (ignorados que no ignoran, archivos trackeados ignorados, guerras de CRLF).

> [!info] Regla de oro: trackeado gana a ignorado
> Un archivo **ya trackeado** ignora `.gitignore` por completo — el ignore solo decide si Git *empieza* a observar archivos nuevos. Por eso aparece la paradoja "añadí el archivo a `.gitignore` y sigue apareciendo en `git status`". El fix requiere des-trackearlo primero (ver Errores comunes).

## `.gitignore`: qué ignora Git

Archivos de texto, uno por línea, que le dicen a Git qué **no** observar (los ignorados no se trackean, no aparecen en `status`, no entran a `add .`). Se commitean: las reglas son del proyecto, no de tu máquina.

### Sintaxis completa

| Patrón | Significado |
|---|---|
| `node_modules/` | **Solo directorios** (la `/` final importa) |
| `build` | Directorio o archivo llamado `build`, en **cualquier** nivel |
| `/build` | `build` solo en la raíz del repo (anclado) |
| `docs/*.pdf` | PDFs dentro de `docs` (un nivel, anclado a `docs`) |
| `**/logs` | `logs` en cualquier profundidad |
| `logs/**/*.log` | `.log` bajo `logs` a cualquier profundidad |
| `!important.log` | **Excepción**: este sí se trackea |
| `*.log` | Comodín de extensión |
| `temp?` | `?` = un solo carácter |
| `\#archivo` | Escape: el `#` es comentario, no patrón |
| `*.md` vs `\#…` | Los espacios finales significan; escapar `!` y `#` cuando son nombres reales |

Comentario = línea con `#`. Línea vacía = no hace nada (sirven para agrupar).

### Las trampas de la negación

```gitignore
# Orden importa — se evalúa de arriba hacia abajo:
build/logs/
!build/logs/debug.log     # ¿funciona? NO si build/logs/ ya está ignorado
```

> [!warning] No se puede re-incluir dentro de un directorio ignorado
> Si un **directorio** está ignorado (`build/logs/`), Git no lo desciende jamás: las excepciones `!` de su interior nunca se evalúan. Verificado por diseño de Git: el patrón final decide por **ruta completa**, y el último que matchea gana. La receta correcta para el caso "ignoro logs salvo uno":

```gitignore
build/logs/*
!build/logs/debug.log     # ahora sí: ignoré el CONTENIDO (con *), no el directorio
```

### Anclado vs no anclado (la diferencia silenciosa)

| Patrón | Ignora en... |
|---|---|
| `doc/frotz/` (con `/` intermedia) | Solo `doc/frotz` **en la raíz** del repo |
| `frotz/` (sin `/`) | Cualquier `frotz` a cualquier nivel |
| `/frotz` | `frotz` solo en la raíz |

Un `/` que no sea el final ancla el patrón — causa frecuente de "mi `.gitignore` funciona en el monorepo raíz pero no en el paquete" (o al revés).

### Fuentes de ignorado: el sistema completo (verificado)

| Fuente | Archivo | ¿Se commitea? | Alcance |
|---|---|---|---|
| `.gitignore` | raíz (y por subdirectorio) | Sí | Todo el que clone — reglas del proyecto |
| `.git/info/exclude` | `.git/info/exclude` | **Nunca** | Solo tu clon (cosas personales de ese repo) |
| `core.excludesFile` | `~/.config/git/ignore` (defecto) | No aplica | Todos tus repos (`.DS_Store`, `.idea/`) |

Las tres fuentes se **aplan** con las mismas reglas de sintaxis y precedencia (verificado: `secreto.txt` en exclude desaparece del `status` local sin afectar a nadie; `core.excludesFile` con `global.txt` igual en todos los repos). La jerarquía de decisión es: patrón más específico/anterior gana, y el último patrón que matchea una ruta decide su destino (ignorada o no). Detalle de configuración en [[Configuración Inicial]].

### Plantilla base multiplataforma

```gitignore
# OS
.DS_Store
Thumbs.db

# Editors
.idea/
.vscode/*
!.vscode/extensions.json
*.swp

# Dependencies
node_modules/

# Build
dist/
build/
*.tsbuildinfo

# Env & secrets
.env
.env.local
*.pem

# Logs
*.log
logs/
```

`.env` en ignore es costumbre de seguridad: los secretos jamás se trackean ni viajan al remoto; si un proyecto necesita valores de ejemplo, se commitea un `.env.example` sin secretos reales y el `.env` real queda siempre ignorado.

## `.gitattributes`: atributos por archivo

Mientras `.gitignore` decide **si** Git observa un archivo, `.gitattributes` decide **cómo** lo trata. Se commitea y gana sobre la config personal de cada dev (`core.autocrlf`) — por eso es la solución definitiva a las guerras de fin de línea.

| Atributo | Valores | Efecto |
|---|---|---|
| `text` | `auto`/`true`/`false` | Normaliza EOL al LF en el índice |
| `eol` | `lf`/`crlf`/`native` | EOL del working tree |
| `-text` | — | Archivo binario: cero conversión |
| `binary` | — | Atajo de `-diff -merge -text` |
| `diff` | driver o `false` | Cómo se difumina (ver drivers abajo) |
| `merge` | driver | Cómo resuelve conflictos (ver drivers abajo) |
| `export-ignore` | — | No incluir en `git archive` (release zips limpios) |
| `export-subst` | — | Sustituye `$Format:...$` en exports |
| `linguist-language`/`linguist-vendored` | — | Tips para GitHub Linguist (stats y color del repo) |
| `working-tree-encoding` | p.ej. `UTF-16` | Codificación distinta en disco |
| `filter` | `lfs` u otros | Filtros externos (Git LFS — ver abajo) |

```gitattributes
# Guerra de EOL terminada para siempre:
*        text=auto eol=lf
*.sh     text eol=lf
*.bat    text eol=crlf
*.png    binary
*.pdf    binary

# Limpieza de releases:
/tests/          export-ignore
/.github/        export-ignore
/.gitignore      export-ignore

# Linguist (cómo GitHub cuenta y colorea el repo):
/vendor/**       linguist-vendored
/docs/**         linguist-documentation
*.blade.php      linguist-language=HTML
```

| Efecto combinado | Resultado en índice | Resultado en disco |
|---|---|---|
| `text=auto` | LF | el que traiga el sistema (o `eol=`) |
| `text eol=lf` | LF | LF siempre (recomendado en equipos mixtos) |
| `text eol=crlf` | LF | CRLF en disco |
| sin nada + `autocrlf` de cada quien | según cada dev | **guerra garantizada** |

Si el repo ya tiene mezcla de EOL: añadir `* text=auto`, y `git add --renormalize .` para renormalizar todo el historial siguiente en un solo commit.

### Drivers de merge y diff personalizados

```ini
# .git/config del repo o .gitattributes + git config
# .gitattributes:
*.lock  merge=ours
```

```bash
git config merge.ours.driver true        # "ours" builtin: resolviendo, siempre queda lo nuestro
git config merge.mine.name "resolución propia"
git config merge.mine.driver "mi-script %O %A %B"
git config diff.mine.textconv "mi-convertidor %O"   # ver diffs de formatos raros convertidos
```

Casos reales: `merge=ours` para locks y archivos generados que jamás deben conflictuar; drivers `textconv` para diffs de SQL/binarios/plantillas; drivers custom para formatos donde el merge textual destruye estructura (SVG complejo, lockfiles con formato propio).

## `.gitmodules` y submódulos

Los submódulos incrustan **otro repo dentro del tuyo** apuntando a un SHA exacto — el `.gitmodules` describe URLs y rutas; el commit del padre fija la versión.

```bash
git submodule add https://github.com/org/lib.git libs/lib    # crea .gitmodules + entrada
git submodule add -b main https://github.com/org/lib.git libs/lib
git clone --recurse-submodules <url>                          # clon completo
git submodule update --init --recursive                       # reparar clon existente
git submodule update --remote libs/lib                        # avanzar al último del remoto
git submodule status                                          # SHA fijado vs actual
git submodule deinit -f libs/lib                               # desregistrar
git rm libs/lib && rm .gitmodules                             # eliminar (con commit)
```

`.gitmodules` (texto plano, se commitea):

```ini
[submodule "libs/lib"]
    path = libs/lib
    url = https://github.com/org/lib.git
    branch = main
```

Flujo de trabajo por SHA (el correcto): el padre fija un commit exacto de la sub-librería. Quien avance la dependencia hace un commit del padre que mueve el SHA — **versionado exacto y auditable**, con la opción de probar ramas con `update --remote` puntual. El dolor clásico: clonar sin `--recurse-submodules` y tener directorios vacíos — el `update --init --recursive` de arriba lo arregla.

### Submodule vs subtree vs monorepo

| | Submodule | Subtree | Monorepo |
|---|---|---|---|
| Modelo | Repo externo con SHA fijado | Código **copiado** al historial del padre | Todo en un repo |
| Tamaño del clon | Chico (sin los subrepos) | Grande (todo el código) | Grande |
| Actualizar deps | Commit del puntero | Merge del historial externo | Editar directo |
| Historia del código externo | Separada, referenciada | **Fusionada** en la del padre | Única |
| Romper al clonar | Frecuente (olvidar `--recurse`) | No | No |
| Versionar release conjunto | Manual (SHA×SHA) | Natural | Natural |
| Uso típico | dotfiles, temas, libs estables | libs que se parchean | empresas (Google, Meta) — necesita tooling (Bazel, Nx) |

Submódulos bien usados (referencias fijas, pocos) vs mal usados (una nube de sub-submódulos) es la diferencia entre trazabilidad y pesadilla. Alternativa sin submódulos: paquetes versionados (npm, PyPI) — ver [[Node JS]].

## `.mailmap`: quién es quién de verdad

Archivo que colapsa identidades duplicadas de autores (cambios de correo, mayúsculas, alias):

```
Juan Pérez <juan@corp.com> <jperez@oldmail.com>
Dev B <devb@corp.com> <dev-b@users.noreply.github.com>
```

Efecto: `git log --format="%an %ae"` y `git shortlog -sne` muestran la identidad unificada y los conteos de contribuciones dejan de estar fragmentados. Obligatorio en repos con historial largo y equipos que rotan de correo.

## `.gitkeep`: el archivo que no es de Git

**No existe en Git**: es un convenio del ecosistema (ver absorbido del vault original) — como Git **no almacena directorios vacíos** (solo trackea blobs dentro de trees), los proyectos que necesitan conservar carpetas vacías (por ejemplo `uploads/`, `fixtures/vacíos/`) colocan un `.gitkeep` (cualquier nombre sirve; el más extendido) para que el directorio viaje en el repo. Su propósito declarado en la práctica: *ser reemplazado o ignorado en cuanto la carpeta tenga contenido real*. No confundir con `.gitignore` (reglas) ni con `.gitkeep` de GitHub Pages (otro caso de uso con Pages).

## Hooks: los ganchos de `.git/hooks/`

Git ejecuta scripts **tuyos** en puntos del ciclo. Viven en `.git/hooks/*.sample` (hay ejemplos) y activan al ser ejecutables (`chmod +x`). **No se commitean** por defecto — por eso nacieron los gestores.

### Catálogo

| Hook | Momento | Uso típico |
|---|---|---|
| `pre-commit` | Antes de crear el commit | Linters, formatters, tests rápidos, bloquear secretos (gitleaks) |
| `prepare-commit-msg` | Al preparar el mensaje | Templates, prefijos de rama (`feat/...:`) |
| `commit-msg` | Al validar el mensaje | ConvCommits, tamaño máximo de línea |
| `post-commit` | Tras el commit | Notificaciones locales |
| `pre-rebase` | Antes de un rebase | Proteger ramas, verificar políticas |
| `pre-push` | Antes del push | Tests completos, verificar que no se pushean tags rotos, bloquear force |
| `post-merge` / `post-checkout` | Tras cambiar de código | Instalar deps si cambiaron (`npm ci`), reconstruir |
| `pre-receive`/`update`/`post-receive` | **Server-side** (el remoto) | Políticas del servidor: probar ramas, validar formato — los de arriba son **client-side** |
| `fsmonitor-watchman` | Cambios en FS | Rendimiento |

> [!warning] Hooks client-side = comodidad, no seguridad
> Se saltan con `git commit --no-verify` o `git push --no-verify` — por eso la validación dura (tests, secretos, firmas) debe repetirse en **CI** y, si el equipo es grande, en hooks server-side de la plataforma (GitHub: branch protection + required checks; ver [[Ramas]]).

### Husky y pre-commit en equipos

Para que los hooks viajen con el proyecto se empaquetan como dependencias:

| Herramienta | Ecosistema | Mecanismo |
|---|---|---|
| **husky** | Node ([[Node JS]]) | `npx husky init` → `.husky/` con scripts versionados |
| **pre-commit** | Python | `.pre-commit-config.yaml` con hooks de comunidad |
| **lefthook** | multi | Alternativa rápida a husky |

```bash
# patrón husky + lint-staged (solo los archivos staged, no todo el repo)
npx husky init
echo "npx lint-staged" > .husky/pre-commit
```

```json
// package.json
{ "lint-staged": { "*.{js,json,md}": ["prettier --write", "eslint --fix"] } }
```

Y en CI el mismo check sin posibilidad de `--no-verify`. Flujo y comandos de staging fino relacionados en [[Comandos Git]]; scripting de los hooks en [[Bash Scripting - Fundamentos Básicos]].

## Sparse-checkout (intro)

Para repos gigantes donde solo necesitás una parte del árbol:

```bash
git sparse-checkout init --cone     # modo cónico (directorios completos)
git sparse-checkout set src/api     # solo esa subcarpeta en el working tree
git sparse-checkout list
git sparse-checkout disable
```

El historial completo está en `.git` (clon normal), pero el disco solo materializa lo elegido — el flujo típico de Linux kernel (`--no-checkout` + sparse) y monorepos enormes.

## Git LFS (pointer)

Git vuela con texto y muere con binarios gigantes (cada versión de un video pesa en el historial **para siempre**). LFS reemplaza archivos grandes por un **pointer** de texto y guarda los binarios en un servidor aparte:

```bash
git lfs install
git lfs track "*.psd"
git lfs track "assets/**/*.mp4"    # genera/actualiza .gitattributes (filter=lfs)
git add .gitattributes && git add archivo.psd
```

`.gitattributes` resultante: `archivo.psd filter=lfs diff=lfs merge=lfs -text`. Los clones necesitan `git lfs pull` (o tener LFS instalado); los CI/CD también. Regla práctica: desde ~50 MB conviene LFS; las fotos de un static site, a veces también. Complementa con `.gitignore` para artefactos de build — ver [[Docker - Docker Compose]] para cómo esto interactúa con contextos de imagen (los `.dockerignore` son el equivalente de Docker).

## CODEOWNERS (pointer de plataforma)

No es un archivo de Git sino de **GitHub** (la plataforma, ver [[Qué es Git]]): `.github/CODEOWNERS` declara dueños por patrón de ruta (`*.js @equipo-frontend`) y la plataforma exige su revisión en PRs. Su primo hermano de Git puro: `pre-rebase`/`pre-push` hooks y `MERGE_HEAD`. No hay nota dedicada en el vault — concepto cubierto aquí para no perder el hilo con la documentación de la plataforma.

## Errores comunes

### 1. Ignoré un archivo ya trackeado y sigue apareciendo

```bash
git rm --cached archivo          # deja de trackearlo, conserva el disco
# o un directorio completo:
git rm -r --cached node_modules
git commit -m "chore: dejar de trackear node_modules"
```

Verificado el flujo conceptual: el ignore recién surtirá efecto porque ahora es "untracked ignorado". La variante *pánico* (`error: the following file has changes staged...`) se resuelve con `-f` consciente o `restore --staged` antes.

### 2. Guerras de CRLF: "todo el archivo aparece modificado"

Síntoma: teammate en Windows edita un archivo, el diff muestra **cada línea** cambiada (solo cambió LF↔CRLF). Secuencia de reparación:

```bash
git config --global core.autocrlf input    # o según tu SO (ver Configuración)
echo "* text=auto eol=lf" >> .gitattributes
git add --renormalize .                    # renormaliza todo el árbol
git commit -m "chore: normalizar EOL"
```

Prevención eterna: `text=auto` en `.gitattributes` del proyecto (gana sobre config personal). Verificado el efecto de `eol` en la tabla combinada arriba.

### 3. El patrón "no ignora nada"

Causas frecuentes: espacio final en la línea (¡significante!), `/` inicial sin querer (ancló a la raíz), negación tapada por un directorio ignorado (arriba), o editar `.git/info/exclude` pensando que era el `.gitignore`. Diagnóstico: `git check-ignore -v ruta/sospechosa` muestra **qué línea de qué archivo** la ignora — el comando que disuelve el 100% de estas dudas.

### 4. Submódulo vacío tras clonar

```bash
git clone --recurse-submodules <url>     # clon correcto
git submodule update --init --recursive  # reparación en clon existente
```

### 5. Commit accidental de secretos

Aunque el `.env` esté en ignore, si alguien hizo `git add -f` o el ignore llegó tarde, **el secreto está en el historial** (ver [[Qué es Git]]: los objetos no se borran solos). Remedios: rotar el secreto **inmediatamente** (siempre), y reescribir historia con herramientas (`git filter-repo`) — que es reescritura de compartida con todas las advertencias de [[Rebase]].

## Cheatsheet

| Tarea | Comando/archivo |
|---|---|
| Ignorar archivos (proyecto) | `.gitignore` commiteado |
| Ignorar solo para mí | `.git/info/exclude` |
| Ignorar en todos mis repos | `core.excludesFile` |
| ¿Por qué se ignora esto? | `git check-ignore -v <ruta>` |
| Dejar de trackear conservando disco | `git rm --cached <f>` (+ commit) |
| Guerra de EOL: fix | `* text=auto` en `.gitattributes` + `git add --renormalize .` |
| Binarios seguros | `*.png binary` en `.gitattributes` |
| Submódulo nuevo | `git submodule add <url> <ruta>` |
| Clon completo | `git clone --recurse-submodules <url>` |
| Estado de submódulos | `git submodule status` / `update --init --recursive` |
| Rastrear grandes archivos | `git lfs track "<patrón>"` |
| Hook pre-commit portable | husky (Node) / pre-commit (Python) |
| Autores unificados | `.mailmap` |
| Release sin basura | `export-ignore` en `.gitattributes` |

## Relacionadas

- [[Configuración Inicial]] — las tres fuentes de ignore, `core.excludesFile`, `core.autocrlf` y activación de hooks
- [[Comandos Git]] — `git rm --cached`, `git add -f`, `git clean -n` y el staging fino que todo esto rodea
- [[Qué es Git]] — por qué los objetos trackeados viven en el historial (y los ignorados, no)
- [[Ramas]] — pre-push/pre-rebase hooks y protección de ramas en flujos de equipo
- [[Node JS]] — husky/lint-staged y el ecosistema npm detrás de los hooks modernos
- [[Bash Scripting - Fundamentos Básicos]] — escribir los propios hooks como scripts
- [[Docker - Docker Compose]] — `.dockerignore` y el efecto de `.gitignore` sobre imágenes y contextos
