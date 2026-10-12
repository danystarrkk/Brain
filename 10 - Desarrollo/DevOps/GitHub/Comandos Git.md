---
tags:
  - desarrollo
  - devops
  - git
tipo: concepto
estado: terminado
---

# Comandos Git

Referencia exhaustiva de los comandos de Git usados a diario, agrupados por tarea, con la matriz de deshacer completa y las reglas de seguridad que separan un atajo de un desastre. Cada flujo crítico está verificado contra Git real; el criterio es entender **qué se mueve** (working tree, índice, HEAD, refs) en cada comando — ver el modelo en [[Qué es Git]].

> [!tip] Regla de lectura de todo comando Git
> Antes de ejecutar, preguntá: *¿esto toca mis archivos, el índice, el puntero de la rama o solo objetos?* Casi todo se reduce a eso. `git status` responde "¿qué hay entre zonas?" y `git restore --staged` deshace la última respuesta.

## Crear y clonar

| Comando | Qué hace |
|---|---|
| `git init` | Crea repo nuevo en el directorio actual (`.git/`) |
| `git init -b main` | Igual pero nace en `main` sin tocar `init.defaultBranch` |
| `git init <dir>` | Crea el repo dentro de `<dir>` |
| `git clone <url>` | Clona con **historial completo** + configura `origin` + tracking |
| `git clone -b rama <url>` | Clona y arranca en esa rama |
| `git clone --depth 1 <url>` | Clon superficial (solo último commit — CI, contenedores) |
| `git clone --recurse-submodules <url>` | Clona también submódulos (ver [[Archivos Especiales]]) |
| `git clone --filter=blob:none <url>` | Clone parcial tipo GitHub (baja blobs bajo demanda) |

```bash
git clone git@github.com:usuario/repo.git        # SSH
git clone https://github.com/usuario/repo.git    # HTTPS
```

## Estado y qué está pasando

| Comando | Qué muestra |
|---|---|
| `git status` | Resumen: staged/unstaged/untracked, rama, adelanto/rezago del upstream |
| `git status -sb` | Corto + rama con `[ahead 2, behind 1]` |
| `git status --porcelain` | Formato máquina (una línea por archivo, sin ruido) — guiones, no humanos |
| `git branch -vv` | Ramas con su upstream y ahead/behind |
| `git remote -v` | Remotos con sus URLs de fetch y push |
| `git stash list` | Pila de stashes (ver [[Stash]]) |

## Staging: construir el próximo commit

El índice (staging) es lo que entra al siguiente commit — ver modelo en [[Qué es Git]]:

| Comando | Qué hace |
|---|---|
| `git add <archivo>` | Pone cambios del archivo (o lo trackea si es nuevo) en el índice |
| `git add .` / `git add -A` | Todos los cambios (del directorio actual / del repo entero) |
| `git add -u` | Solo cambios y borrados de archivos **ya trackeados** (ignora untracked) |
| `git add -p` | **Patch interactivo**: hunk por hunk, sí/no/partir — commits quirúrgicos |
| `git add -i` | Menú interactivo completo (patch, add, revert, update, stash) |
| `git add -e` | Abre el diff en el editor para decidir manualmente qué entra |
| `git restore --staged <archivo>` | Saca el archivo del índice (vuelve al estado del HEAD) — **no toca tus archivos** |
| `git rm --cached <archivo>` | Deja de trackear el archivo pero **lo conserva en disco** (para ignorarlo, ver [[Archivos Especiales]]) |
| `git mv <viejo> <nuevo>` | Renombra y deja el cambio staged (equivale a mv + add + add del viejo) |

```bash
git add -p     # por hunk: y (sí), n (no), s (partir), e (editar a mano)
```

`git add -p` es la diferencia entre "commiteé medio archivo por error" y commits limpios: parte un hunk con `s` y edita a mano con `e` para mandar solo lo que corresponde.

## Commits

| Comando | Qué hace |
|---|---|
| `git commit -m "msg"` | Commit con el índice actual |
| `git commit -am "msg"` | Añade **solo archivos ya trackeados** con cambios y commitea — **no commitea untracked** |
| `git commit` (sin -m) | Abre `core.editor` para mensaje (primeras líneas: asunto; línea en blanco; cuerpo) |
| `git commit --amend` | **Reemplaza** el último commit con el índice actual (nuevo SHA) |
| `git commit --amend -m "nuevo"` | Amenda solo el mensaje |
| `git commit --amend --no-edit` | Amenda contenido sin tocar el mensaje |
| `git commit --amend --reset-author` | Amenda y reasigna autoría (tras cambiar `user.email`) |
| `git commit --fixup <sha>` | Crea commit marcado para fusionarse con `<sha>` en `rebase -i --autosquash` |
| `git commit --squash <sha>` | Igual pero propone mensaje combinado |
| `git commit --allow-empty -m "marker"` | Commit sin cambios (marcadores de release/CI) |
| `git revert <sha>` | **Commit nuevo** que deshace los cambios de `<sha>` — seguro en ramas compartidas |

Convención de mensajes (ConvCommits): `tipo(ámbito): descripción imperative` — `feat(auth): añade login OAuth`, `fix(api): corrige 500 en /users`. El tipo alimenta changelogs automáticos.

> [!warning] `--amend` y `--force` son la misma puerta
> `--amend` crea un commit **distinto** con el mismo mensaje. Si ya lo hiciste `push`, el remoto tiene el viejo: necesitás `--force-with-lease` (ver [[Ramas]]). Regla: **amend solo commits no publicados**. El escrúpulo típico ("¿y si lo hice publicado?") se resuelve con `git revert`.

## Inspección: log, show, diff

### `git log` — el historial con filtros

```bash
git log                                   # historial completo (q para salir)
git log --oneline                         # una línea por commit
git log --graph --oneline --decorate --all # grafo de ramas/tags — el favorito de equipos
git log --follow archivo.txt              # historial aunque hayan renombrado el archivo
git log -p archivo.txt                    # con diffs
git log --stat                            # con resumen de archivos tocados
git log --since="2 weeks ago" --until=today
git log --author="Nombre"                 # por autor
git log --grep="fix"                      # por mensaje
git log HEAD~5..HEAD                      # últimos 5 commits
git log -S "texto_funcion"                # "pickaxe": commits que añaden/ quitan esa cadena
```

Formatos personalizados (útiles para changelogs, verificado):

```bash
git log --format="%h %an %ad %s" --date=short
# 68b0aff T 2026-10-12 c1

git log --format="%C(yellow)%h%Creset %s %C(cyan)[%an]%Creset"
# 68b0aff c1 [T]
```

### `git diff` — la matriz completa

La confusión #1: `git diff` sin argumentos compara **working tree vs índice**. Hay 6 combinaciones útiles:

| Comando | Compara | Uso típico |
|---|---|---|
| `git diff` | working tree ↔ **índice** | ¿qué edité y todavía no add-ee? |
| `git diff --staged` (o `--cached`) | índice ↔ **HEAD** | ¿qué voy a commitear? |
| `git diff HEAD` | working tree ↔ HEAD | "todo lo que hice" (staged + unstaged) |
| `git diff <rama>` | working tree ↔ puntero de esa rama | ¿me diferencio de main? |
| `git diff <rama>...HEAD` | rama ↔ HEAD desde el ancestro común | la diferencia **real** de tu rama |
| `git diff <sha1> <sha2>` | esos dos commits | comparar cualquier par |
| `git diff <rama1> <rama2>` | punteros de dos ramas | qué traerá el merge |

Flags útiles: `--stat` (resumen), `--name-only` (solo archivos), `--word-diff` (palabras resaltadas), `-U1` (contexto mínimo), `-- path` (limitar rutas).

### Otras ventanas de inspección

| Comando | Qué hace |
|---|---|
| `git show <sha>` | Todo de ese commit: mensaje + diff (también funciona con tags — ver [[Tags]]) |
| `git show HEAD~2:archivo` | Contenido del archivo **tal como estaba** en ese commit |
| `git blame -L 10,25 archivo` | Quién escribió cada línea del rango (con fecha) |
| `git blame -w archivo` | Ignora cambios de espacios/blancos (menos ruido) |
| `git blame -C archivo` | Detecta líneas movidas entre archivos |
| `git log -L 10,25:archivo` | Historial de evolución de ese rango de líneas |

### `git bisect`: búsqueda binaria de la regresión

Cuando algo se rompió entre hace 200 commits:

```bash
git bisect start HEAD HEAD~50   # HEAD = malo conocido, HEAD~50 = bueno conocido
git bisect good <sha>           # marcar el commit actual como bueno
git bisect bad                  # marcar como malo
git bisect reset                # salir del modo bisect
```

Git salta al commit intermedio, probás, marcas `good`/`bad`, y en `log2(50)≈6` iteraciones encontrás el culpable. Verificado el ciclo `bisect start` + `bisect log`. Automatizado:

```bash
git bisect start HEAD HEAD~50
git bisect run npm test         # o ./test.sh: exit 0 = good, 1-124/125-127 = bad
git bisect reset
```

Con `run`, el script es el juez — imprescindible para regresiones reproducibles.

## Moverse por el historial

| Comando | Qué hace |
|---|---|
| `git checkout <rama>` | Cambia HEAD a la rama (con tracking implícito si existe) |
| `git checkout -b <rama>` | Crea y entra |
| `git checkout -b <rama> <base>` | Crea desde el commit/rama que elijas |
| `git checkout <sha>` | **Detached HEAD**: navegar suelto (ver [[Ramas]]) |
| `git checkout <sha> -- <archivo>` | Saca el archivo **de ese commit** al working tree (sin mover HEAD) |
| `git switch <rama>` | Igual que checkout para cambiar de rama (más claro, git ≥2.23) |
| `git switch -c <rama>` | Crea e entra (equivale a checkout -b) |
| `git switch -` | Vuelve a la rama anterior |
| `git switch --detach <rama>` | Navega sin mover la rama |
| `git restore <archivo>` | Descarta cambios del working tree (usa el índice) |
| `git restore --source=HEAD~2 <archivo>` | Restaura el archivo desde cualquier commit |

`switch`/`restore` (git 2.23+) separan lo que `checkout` mezclaba (cambiar de rama vs descartar cambios vs extraer archivos). La distinción importa porque **el mismo comando con argumentos distintos hacia cosas muy distintas**, y la gente perdía trabajo con `checkout -- .`.

## La matriz de deshacer (la referencia más importante)

Cada "deshacer" resuelve un problema distinto. Elegir mal = pérdida de trabajo o de contexto:

| Situación | Comando | Efecto en working tree | Efecto en índice | Efecto en HEAD/rama | ¿Recuperable? |
|---|---|---|---|---|---|
| Saqué un archivo al staging por error | `git restore --staged <f>` | nada | vuelve a HEAD | nada | sí (los cambios siguen en disco) |
| Edité un archivo y quiero el estado del índice | `git restore <f>` | **descarta cambios** | nada | nada | **no** (cambios perdidos salvo reflog del índice) |
| Edité y quiero el estado del último commit | `git restore --source=HEAD <f>` | sobreescribe con el commit | nada | nada | no |
| Quiero deshacer el último commit **conservando** cambios | `git reset HEAD^` (mixed, defecto) | conserva todo | limpia el índice | rama retrocede 1 | sí |
| Quiero deshacer commit dejándolo todo staged | `git reset --soft HEAD^` | conserva | conserva staged | rama retrocede 1 | sí |
| Quiero **borrar** el trabajo del commit (peligroso) | `git reset --hard HEAD^` | **borra cambios** | limpia | rama retrocede | vía reflog |
| El commit ya se publicó | `git revert <sha>` | crea commit inverso | — | avanza | seguro para siempre |
| Commiteé con mensaje malo y no está publicado | `git commit --amend -m "..."` | nada | — | reemplaza el commit | sí |
| "Esto ya estuvo bien antes" | `git reflog` → `git reset --hard HEAD@{n}` | restaura | restaura | repunta | la red de seguridad total |

`HEAD^` = el padre de HEAD; `HEAD~3` = tres hacia atrás por primera generación. `--hard` **también** pisa archivos sin confirmar de otras tareas — con cambios sucios sin stash, puede borrar trabajo ajeno al commit.

> [!warning] `reset --hard` no borra objetos, borra punteros
> Los commits siguen en el object store; el reflog los conserva alcanzables ~90 días (`gc.reflogExpire`). Recuperación inmediata: `git reflog`, identificar el SHA, `git reset --hard HEAD@{n}` o `git switch -c recuperada <sha>`. **Casi nada está perdido si fue commiteado (o incluso solo stash-eado) alguna vez** — ver sección de recuperación abajo.

### `git reflog`: la máquina del tiempo de emergencia

Cada movimiento de HEAD (commit, reset, checkout, rebase, merge, stash) deja una entrada:

```
$ git reflog
e0e3a63 HEAD@{0}: commit: c3
eca668c HEAD@{1}: commit: c2-perdido
52b952a HEAD@{2}: branch: Created from HEAD
```

Verificado: aun tras "perder" un puntero de rama (movido el ref a un commit viejo), `git reflog show feature-loss` muestra la secuencia completa y `git update-ref refs/heads/feature-loss feature-loss@{1}` repunta la rama al commit perdido. Los commits no se pierden por accidente: se pierden **punteros**, y el reflog recuerda cada puntero anterior. Detalle completo en [[Ramas]] (recuperación de ramas) y [[Stash]] (recuperación de stashes borrados).

## Mover y borrar archivos

| Comando | Efecto |
|---|---|
| `git mv a b` | Renombra (staged) — más rápido y claro que mv + add |
| `git rm a` | Quita del disco y del índice |
| `git rm --cached a` | Quita del índice, **conserva el archivo** (para ignorarlo) |
| `git rm -r --cached dir/` | Deja de trackear un directorio entero (ej: `node_modules` commiteado por error) |
| `git clean -n` | **Dry-run**: qué untracked se borraría (siempre primero) |
| `git clean -fd` | Borra untracked y directorios — tras `-n` y con cuidado |
| `git clean -ndx` | Borra también los ignorados (limpieza total, muy destructivo) |

> [!warning] `git clean` es el único que borra **objetos sin respaldo**
> Todo lo demás deja rastro en reflog; `clean` mata archivos que Git nunca vio. Protocolo: `-n` siempre, y mejor `git stash push -u` (ver [[Stash]]) que `clean`.

## Deshacer con contexto: `git stash` y worktrees

Para saltar a otra cosa sin perder trabajo sucio:

```bash
git stash push -u -m "wip: formulario"   # guarda tracked + untracked
git stash pop                             # recupera y quita de la pila
git worktree add ../repo-fix rama-fix     # segundo directorio con la misma rama
```

Referencia completa en [[Stash]] y [[Ramas]] (worktrees). Los **temp commits** (commits rotulados `wip` para no perder contexto) son la alternativa cuando el stash crece demasiado.

## Verificar: hashes y rangos

| Forma | Significado | Para qué |
|---|---|---|
| `abc1234` | Prefijo del SHA completo (40 hex) | Identificar commits (Git resuelve prefijos únicos) |
| `HEAD~3` | 3 padres hacia atrás | Navegación relativa |
| `abc1234^` / `abc1234^2` | Padre / segundo padre (merges) | Navegar grafos |
| `A..B` | Commits en B **no** en A | "¿Qué trae mi rama?" (`git log main..HEAD`, `git log HEAD..origin/main`) |
| `A...B` | Commits en **cualquiera** de los dos pero no en ambos | Diferencia simétrica (`git log main...feature`, `git diff main...feature`) |

Verificado en la práctica: con `main` teniendo `c3` y `feat` divergiendo con `feat c`, `main..feat` lista solo `feat c`, y `main...feat` lista `c3` **y** `feat c` (los dos lados de la divergencia). Regla: **`..` = "qué tiene B que yo no"** (apuntar el cañón); **`...` = "en qué nos diferenciamos"** (ambos lados).

## Flujo remoto mínimo

| Comando | Qué hace |
|---|---|
| `git fetch` | Actualiza `refs/remotes/origin/*` — **nunca** toca tu código |
| `git fetch -p` | Lo mismo y borra refs de ramas remotas eliminadas |
| `git pull` | `fetch` + `merge` (o `rebase` si `pull.rebase=true` — ver [[Configuración Inicial]]) |
| `git pull --rebase` | Integra tu trabajo encima de lo nuevo — historial lineal |
| `git push` | Empuja la rama actual a su upstream (`push.default` decide, ver [[Configuración Inicial]]) |
| `git push -u origin rama` | Empuja y fija el upstream la primera vez |
| `git push --force-with-lease` | Sobreescribe el remoto **solo si nadie avanzó desde tu último fetch** — ver [[Ramas]] |

Detalle de seguridad de `--force-with-lease` en [[Ramas]] y [[Rebase]] (rebase de ramas publicadas). Flujo completo de PRs es de la plataforma (GitHub), no de Git — ver [[Qué es Git]].

## Telemetría: Git no reporta a nadie

Git **no envía datos de uso**: no hay phone-home, ni analytics, ni identificadores anónimos. Las únicas comunicaciones salientes son hacia los remotos que vos configurás (SSH/HTTPS) y, opcionalmente, actualizaciones del propio programa (`git upgrade`, opt-in). Los hooks son scripts tuyos (ver [[Archivos Especiales]]). Cualquier telemetría en tu flujo viene de la **plataforma** (GitHub/GitLab) o de tu CI, no del binario `git`.

## Errores comunes (síntoma → arreglo)

| Síntoma | Causa | Arreglo |
|---|---|---|
| `Please move or remove them before you can merge` | Archivos sin trackear chocan con el merge | `git stash push -u` o moverlos, reintentar |
| `error: pathspec 'master' did not match any file(s) known to git` | La rama se llama `main` (defecto moderno) | `git switch main` o `git checkout -b main` |
| `detached HEAD` + miedo | Estás en un commit suelto | `git switch <rama>` para volver, o crear rama para conservar |
| `! [rejected] non-fast-forward` | El remoto avanzó desde tu último fetch | `git pull --rebase` (o merge) y reintentar push |
| Commiteé un archivo gigante / secreto | Entró al índice | Para lo inmediato: `git rm --cached <f>`; si ya se publicó, la historia debe reescribirse (ver [[Rebase]]) + rotar el secreto **siempre** |
| `git commit` se abre en vim y no sé salir | `core.editor` es vim | `:wq` para guardar, `:q!` para abortar (ver [[Vim Comandos Esenciales]]) |
| `Use -f if you really want to add them` | `git add` rechaza archivos ignorados | Si es deliberado: `git add -f`; si no, revisá la regla de ignore (ver [[Archivos Especiales]]) |
| `stale index.lock` tras un crash | Quedó el lock de una operación interrumpida | Verificar que no haya git corriendo; borrar `.git/index.lock` a mano (último recurso) |
| `did not match any files` con `git rm` | El archivo ya no existe en disco pero sí en el índice | `git rm --cached <f>` o `git restore --staged` |

## Cheatsheet de bolsillo

| Tarea | Comando |
|---|---|
| Ver estado | `git status -sb` |
| Ver historial | `git log --graph --oneline --decorate --all` |
| Staging fino | `git add -p` |
| Deshacer staging | `git restore --staged <f>` |
| Commit rápido | `git commit -am "msg"` (solo trackeados) |
| Arreglar último commit | `git commit --amend --no-edit` (¡no publicado!) |
| Deshacer commit sin perder cambios | `git reset HEAD^` |
| Ver qué voy a commitear | `git diff --staged` |
| Ver todo lo que hice | `git diff HEAD` |
| Quién escribió esta línea | `git blame -L a,b <f>` |
| Buscar la regresión | `git bisect start HEAD HEAD~N` + `good`/`bad` |
| Guardar trabajo sucio | `git stash push -u` |
| Recuperar lo borrado | `git reflog` → `git reset --hard HEAD@{n}` |
| Sync seguro | `git fetch -p && git pull --rebase` |
| Deshacer un commit publicado | `git revert <sha>` |

## Relacionadas

- [[Qué es Git]] — el modelo (objetos, índice, HEAD) que hace coherentes todos estos comandos
- [[Configuración Inicial]] — `push.default`, `pull.rebase`, alias, editores y credenciales
- [[Ramas]] — todo lo de ramas: merge, conflictos, tracking, flujos y recuperación
- [[Rebase]] — reescribir historia: `-i`, autosquash y la red de seguridad ORIG_HEAD
- [[Stash]] — la referencia completa del atajo para trabajo sucio
- [[Tags]] — marcar releases sobre estos comandos
- [[Archivos Especiales]] — ignores, atributos, hooks que interceptan estos flujos
- [[Comandos Básicos de Sistema y Red en Linux]] — el terreno de shell sobre el que vive Git
- [[Vim Comandos Esenciales]] — para sobrevivir al editor por defecto de Git
