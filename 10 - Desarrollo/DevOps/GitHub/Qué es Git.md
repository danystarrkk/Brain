---
aliases: ["Git", "Control de Versiones", "Sistema de Control de Versiones"]
tags:
  - devops/git
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Configuración Inicial]]"
  - "[[Comandos Git]]"
  - "[[Ramas]]"
  - "[[Rebase]]"
  - "[[Tags]]"
  - "[[Qué es una API]]"
fuente: "https://git-scm.com/about"
---

# Qué es Git

**Git** es un sistema de control de versiones (VCS) **distribuido**: guarda el historial completo de un proyecto como una cadena de instantáneas inmutables, permitiéndote viajar a cualquier punto del pasado, ramificar sin costo y trabajar sin conexión. Fue creado por Linus Torvalds en 2005 para desarrollar el kernel de Linux y hoy es el estándar de facto de la industria — desde [[Qué es una API|APIs]] y frontends hasta firmware.

La forma mental correcta de verlo es una **máquina del tiempo**: cada commit es un instante al que puedes volver con seguridad. Esa es la promesa fundacional:

![[Pasted image 20250125071209.png]]

> [!info] La pregunta que responde Git
> *¿Cómo mantengo el estado exacto de mi proyecto en cualquier momento, permito que 100 personas trabajen a la vez sobre el mismo código y no pierdo nada cuando algo sale mal?* Cada componente de Git responde a una parte de esa pregunta: los **objetos** guardan el estado, las **refs** lo apuntalan, el **índice** decide qué entra en el siguiente snapshot, y el modelo **distribuido** replica todo en cada máquina.

## VCS distribuido vs centralizado

En un VCS centralizado (CVCS) como SVN, CVS o Perforce existe **un** servidor con el historial; los clientes solo tienen una copia del árbol actual. En Git **cada clon es un repositorio completo** con todo el historial:

| Característica | Centralizado (SVN, CVS, Perforce) | Distribuido (Git, Mercurial) |
|---|---|---|
| Dónde vive el historial | Solo en el servidor | En **cada clon** |
| Trabajar sin conexión | No (salvo rarezas) | Sí: commit, branch, merge, blame offline |
| Commits baratos | No: cada commit viaja a la red | Sí: el commit es local y atómico |
| Velocidad | Latencia de red en cada operación | Todo lo local es instantáneo |
| Puntos únicos de falla | El servidor | Ninguno: cada clon es backup |
| Ramas | Caras y conflictivas (copies de carpetas) | Baratas: apuntar un puntero de 41 bytes |
| Flujo de trabajo típico | trunk + ramas grandes y longevas | ramas efímeras, integración frecuente |
| Historia reescribible | Complicado (svns) | Sí, antes de compartir (ver [[Rebase]]) |
| Ejemplos hoy | SVN en herencia, Perforce en game-dev | Git, Mercurial (Google, Mozilla) |

Consecuencia práctica del modelo distribuido: **no existe el concepto de "repositorio central" en Git**. Git solo conoce tu repositorio local y un número arbitrario de *remotos* (típicamente `origin`). Que un remoto sea "el oficial" es una **convención del equipo**, no una regla de Git. Cada integrante tiene una copia completa, así que el equipo entero sobrevive a la pérdida del servidor, y sincronizan por empuje (`git push`) y tirón (`git fetch` / `git pull`).

En Git tampoco existe el término *proyecto*: la unidad es el **repositorio** (directorio con un `.git/` dentro).

## Modelo de objetos: todo es contenido direccionable

Git no guarda "archivo A cambió 3 líneas". Guarda **objetos** identificados por un hash SHA-1 (SHA-256 en repositorios modernos, ver `git init --object-format=sha256`) de su contenido exacto. Dos archivos con el mismo contenido son **un solo objeto** compartido (deduplicación), y un archivo sin cambios no se vuelve a guardar nunca.

Hay exactamente **cuatro tipos de objeto**:

| Tipo | Contenido | ¿Referencia a otros objetos? | Ejemplo de uso |
|---|---|---|---|
| **blob** | Los bytes crudos de un archivo (sin nombre, sin ruta) | No | Cada versión de cada archivo trackeado |
| **tree** | Directorio: lista de entradas (nombre, modo, tipo, SHA) | Sí → blobs y subtrees | El "snapshot" de un directorio |
| **commit** | Metadatos + puntero al tree raíz + padre(s) | Sí → 1..N commits + 1 tree | El instante de la máquina del tiempo |
| **tag** (anotado) | Objeto firmado: tagger, mensaje, puntero a commit | Sí → 1 commit | Release `v2.1.0`, ver [[Tags]] |

Verificado en un repo de prueba (`git cat-file -t`):

```bash
$ git cat-file -t HEAD          # el commit actual
commit
$ git cat-file -t HEAD^{tree}   # el snapshot completo del proyecto
tree
$ git cat-file -t v1.0          # un tag ANOTADO es un objeto propio
tag
```

El mismo archivo idéntico en dos carpetas o en dos commits genera **un único blob**: Git deduplica por contenido, no por ruta. Por eso clonar un repo enorme es rápido si el histórico tiene pocos cambios reales (y por eso la historia crece poco con cambios pequeños — pero crece mucho si cambian archivos grandes, ver LFS en [[Archivos Especiales]]).

> [!tip] Instantáneas, no diffs
> Git guarda el **estado completo** del proyecto en cada commit (vía el tree), no una lista de diferencias. Los diffs que ves en `git log -p` se **calculan al vuelo** comparando dos snapshots. Esto explica operaciones sorprendentemente baratas: `git checkout` a un commit viejo es "reescribir los archivos del tree marcado", y `git branch nueva` ni siquiera toca archivos — solo crea un archivo de texto de 41 bytes en `.git/refs/heads/`.

> [!warning] Los objetos son inmutables pero los punteros no
> El hash de un commit cubre su contenido **y el de todos sus ancestros** (por eso `git commit` incluye un campo `parent`). Cambiar un byte de un commit antiguo produce un SHA distinto y rompe todos los descendientes. Esa cadena de hashes es la integridad criptográfica de Git: si alguien edita el historial a mano, todos los SHA posteriores dejan de coincidir y Git lo detecta.

### Anatomía de un commit (verificada)

Un commit es un objeto de texto con esta forma exacta (`git cat-file commit HEAD`):

```
tree 40ce32801aa3d5001925dc4dc342210ec3b57b2e
author T <t@t> 1791782011 -0500
committer T <t@t> 1791782011 -0500

c1: add a
```

| Campo | Significado |
|---|---|
| `tree` | Snapshot del proyecto completo en ese instante (directorio raíz) |
| `parent` | SHA del commit padre (ausente en la raíz; dos o más en merge) |
| `author` | Quién escribió el cambio (fecha + offset horario incluidos) |
| `committer` | Quién registró el cambio (igual al author salvo en rebases/merges con otra persona) |
| mensaje | Texto libre tras línea en blanco |

Detalle que explica muchos misterios: **author ≠ committer**. Cuando tu commit pasa por un merge de alguien o por `git rebase` con committer distinto, el `author` se conserva (tu autoría) pero el `committer` cambia (quien integró). Por eso `git log --format="%an %cn"` a veces muestra dos nombres distintos.

### Commits con dos padres: el merge

Un commit de merge tiene **dos (o más) padres**, lo que forma un grafo (DAG), no una línea recta:

```bash
$ git log --graph --oneline --all
*   1a492bc merge resolved      <- 2 padres: 9a290b6 y 6248c97
|\
| * 6248c97 feat
* | 9a290b6 main work
|/
* 63e7c1d base
```

Verificado en la práctica: `git merge feature` en conflicto produce un commit con ambos padres cuando resuelves. La rama `feature` es simplemente un puntero al commit `6248c97`; el merge es un nuevo nodo con dos flechas hacia atrás. Todo lo que ves en `git log --graph` es este DAG.

## Las tres áreas: working tree, índice y HEAD

Todo comando de Git (ver [[Comandos Git]]) actúa sobre — o mueve datos entre — **tres zonas**:

```
   edite archivos        git add              git commit
Working Tree  ────────▶  Index (staging)  ──────────────▶  Repositorio (.git/objects)
 (tus archivos)          (snapshot candidato)              (historial inmutable)
        ▲                                                                │
        └────────────────────── git checkout / restore ──────────────────┘
```

| Área | Qué es | Dónde vive | Contenido |
|---|---|---|---|
| **Working tree** | Tus archivos editables | El directorio del repo | Estado actual, sin garantías |
| **Index** (staging) | Snapshot candidato del próximo commit | `.git/index` (binario) | Qué cambios entran en el siguiente commit |
| **Repositorio** | Historial completo de objetos y refs | `.git/objects/`, `.git/refs/` | Commits, trees, blobs, tags inmutables |

Por eso hay dos comandos de "deshacer": `git restore <archivo>` descarta cambios del working tree respecto del índice, y `git restore --staged <archivo>` devuelve el índice al estado del último commit. El índice es la pieza que hace único a Git entre los VCS: te permite construir commits **a partir pieza por pieza** (ver `git add -p` en [[Comandos Git]]).

## Refs: cómo Git encuentra los commits

Los commits se guardan por hash — encontrar "la última versión" sin un nombre sería imposible. Ahí entran las **refs** (referencias): archivos de texto de 41 bytes que contienen un SHA. Su ubicación es exacta:

| Ref | Archivo | Ejemplo de contenido |
|---|---|---|
| Rama actual | `.git/HEAD` | `ref: refs/heads/main` |
| Rama | `.git/refs/heads/main` | `e06cbcddaba2a8ffe948085ad1d4d579d4618dfb` |
| Tag anotado | `.git/refs/tags/v1.0` | `597f800ac091d49ab3a7e949eca9c1cf86e4cb8d` |
| Remoto | `.git/refs/remotes/origin/main` | último SHA visto de `main` en el remoto |
| Stash | `.git/refs/stash` (vía reflog) | ver [[Stash]] |

Verificado: `cat .git/HEAD` imprime `ref: refs/heads/main`. **Crear una rama es crear ese archivo** — por eso `git branch` es instantáneo y "gratis" en Git, mientras que en SVN duplicar la carpeta del proyecto costaba minutos y gigas. Mover la rama = sobreescribir 41 bytes. Borrar la rama = borrar el archivo (los objetos quedan y el reflog permite recuperarlos, ver [[Ramas]]).

> [!info] `HEAD` es solo "dónde estoy parado"
> `HEAD` es una ref que apunta a la rama actual (o directamente a un commit cuando estás en *detached HEAD*). `git commit` mueve el puntero de tu rama a un commit nuevo; `git checkout` mueve `HEAD`. Entender que **Git es un grafo de objetos + punteros que se mueven** explica el 90% de los comandos y de los sustos (ver [[Ramas]] para detached HEAD y [[Comandos Git]] para la matriz de deshacer).

## Modelo de trabajo distribuido

El flujo típico con un remoto (normalmente GitHub):

```
              git push                       git fetch / git pull
tu repo local ──────────▶ remoto (origin) ─────────────────────────▶ refs/remotes/origin/*
   commits locales         rama main en el servidor                 copia local del estado del remoto
```

- `git clone <url>`: copia **todo** el historial + configura `origin` + crea rama local con *tracking*.
- `git fetch`: baja objetos y actualiza `refs/remotes/origin/*` **sin tocar tu código** (seguro siempre).
- `git push`: sube tus commits y mueve la rama del remoto (rechaza si alguien avanzó — necesita `--force` para sobreescribir, ver [[Ramas]] y [[Rebase]]).
- `git pull` = `fetch` + `merge` (o `rebase`, según `pull.rebase`, ver [[Configuración Inicial]]).

Cada desarrollador tiene una copia completa, así que un equipo puede trabajar sin conexión y resolver conflictos al integrar — y la pérdida total de un disco nunca significa la pérdida del proyecto.

## Git vs otras herramientas

| | Git | SVN | Mercurial (Hg) | VSS/CVS |
|---|---|---|---|---|
| Modelo | Distribuido | Centralizado | Distribuido | Centralizado |
| Identidad de cambios | SHA-1/SHA-256 del contenido | Número de revisión global | Hash + revlog | Número correlativo |
| Historia | Reescribible localmente | Prácticamente inmutable | Reescribible (obscurecida) | Inmutable |
| RAMAs | Punteros baratas | Copies en servidor | Punteros baratas | Copies caras |
| Curva de aprendizaje | Alta (modelos internos) | Media | Baja ("Git-friendly") | Baja |
| Industria (2026) | Estándar | Legado | Nicho (Google usó su fork) | Extinto |

Mercurial merece mención: es técnicamente comparable a Git (también distribuido, también hashes) pero eligió simplicidad y comandos más amables; perdió la guerra de adopción no por técnica sino por el ecosistema (GitHub, CI, tooling) que se construyó alrededor de Git.

## Historia breve

| Año | Evento |
|---|---|
| 1991–2002 | El kernel de Linux se desarrolla con **BitKeeper**, un VCS propietario |
| Abril 2005 | BitKeeper retira la licencia gratuita tras la disputa con el proyecto; **Linus Torvalds** escribe Git en ~10 días para cubrir el hueco (primera versión: 3 abr 2005, commit inicial `7e069a3`) |
| 2005 | **Junio Hamano** asume como maintainer principal de Git — sigue siéndolo décadas después, uno de los papeles de maintainer más longevos del open source |
| 2008 | Nace **GitHub** como plataforma de hosteo con pull requests |
| 2015 | Git 2.5 introduce `git worktree`; el proyecto ronda 20 comandos "plenos" y ~150 con alias |
| 2022+ | SHA-256 madura (`--object-format`), transición lenta por compatibilidad de la industria |

Diseño original (que sigue vigente): **velocidad** (todo local), **simplicidad del modelo** (branch = archivo, commit = objeto), **ramificación barata**, **integridad total** (hashing) y **divergencia controlada** (merges explícitos).

## Git (la herramienta) vs GitHub (la plataforma)

Confusión clásica: son cosas distintas.

| | Git | GitHub |
|---|---|---|
| Qué es | Herramienta de línea de comandos local | Servicio web que aloja repos Git |
| Funciona sin internet | Sí | No (el hosting es su razón de ser) |
| Pull requests | No existen (el concepto es merge) | Mecanismo de revisión + merge de la plataforma |
| Issues, Actions, Pages, Packages | No | Sí — son features de la plataforma, no de Git |
| Quién lo controla | Proyecto open source (Junio Hamano) | Microsoft |
| Alternativas | — | GitLab, Gitea, Bitbucket, Codeberg |

Commitear, ramificar, hacer rebase o tag es **Git**. Abrir un pull request, revisar con CODEOWNERS, correr CI en Actions o publicar un release es **GitHub** (el mismo modelo lo ofrecen GitLab o Gitea). Todo lo que haces en local funciona igual con cualquier host o sin host.

Errores conceptuales frecuentes al empezar:

1. **"Perdí el archivo, está perdido para siempre"** — si hiciste commit, el objeto está en `.git/objects`; solo perdiste el *puntero*. `git reflog` lo encuentra (ver [[Comandos Git]]).
2. **"Git guarda diffs"** — Git guarda snapshots; los diffs son calculados.
3. **"El remoto es el repositorio y lo mío es una copia de trabajo"** — tu clon es un repositorio de segunda clase completo; el remoto es igual de pobre o rico que tú.
4. **"Borrar una rama borra su trabajo"** — borra 41 bytes; los commits quedan alcanzables vía reflog hasta el GC.

## Cheatsheet conceptual

| Concepto | Respuesta corta |
|---|---|
| ¿Qué guarda Git? | Snapshots completos (trees), no diffs |
| ¿Cuántos tipos de objeto? | 4: blob, tree, commit, tag (anotado) |
| ¿Qué es un hash? | SHA-1 (o SHA-256) del contenido del objeto |
| ¿Dónde vive una rama? | `.git/refs/heads/<nombre>` = un SHA de 41 bytes |
| ¿Qué es HEAD? | Ref que apunta a la rama actual (o a un commit suelto) |
| ¿Qué es el índice? | El snapshot candidato al próximo commit (staging) |
| ¿Cuántos padres tiene un commit normal? | 1 (la raíz: 0; un merge: 2+) |
| ¿Qué es origin? | Convención de nombre del remoto principal — no hay remoto "central" en Git |
| ¿Qué es distribuido? | Cada clon tiene el historial completo y puede trabajar offline |
| ¿Author vs committer? | Quién escribió el cambio vs quién lo registró |

## Errores comunes

1. **Confundir repositorio local con remoto** — `git push` no es "guardar": es sincronizar con un remoto específico.
2. **Tocar `.git/` a mano** — `.git/index`, `refs` y `objects` tienen invariantes; manipularlos a mano corrompe el repo (recuperación de emergencia: `git fsck`).
3. **Esperar que `git status` "encuentre" archivos sin commit** — sin `.gitignore` el ruido de `node_modules` etc. satura el estado (ver [[Archivos Especiales]]).
4. **Usar un solo branch gigante por miedo a ramas** — el modelo de Git está optimizado para ramas efímeras baratas; el miedo a ramas es síndrome de SVN (ver [[Ramas]]).

## Relacionadas

- [[Configuración Inicial]] — configurar usuario, ramas por defecto y firmas antes del primer commit
- [[Comandos Git]] — la referencia exhaustiva de comandos: staging, diff, deshacer, reflog
- [[Ramas]] — el grafo en acción: merge, tracking, conflictos, flujos de equipo
- [[Rebase]] — reescribir historia: por qué es potente y por qué es peligroso en compartidas
- [[Stash]] — el atajo para guardar trabajo sucio sin commit
- [[Tags]] — marcadores inmutables para releases sobre el modelo de objetos
- [[Archivos Especiales]] — `.gitignore`, `.gitattributes`, submódulos y hooks
- [[Qué es una API]] — contexto: Git y sus plataformas son la base del flujo moderno de entrega de software
