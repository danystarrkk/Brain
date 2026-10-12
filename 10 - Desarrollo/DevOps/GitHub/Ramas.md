---
tags:
  - desarrollo
  - devops
  - ramas
tipo: concepto
estado: terminado
---

# Ramas

Una rama en Git es un **puntero movable de 41 bytes** que señala a un commit. Nada más: crearla, moverla y borrarla es instantáneo y gratis, porque no duplica archivos — solo escribe un SHA en `.git/refs/heads/<nombre>`. Ese diseño "barato" es lo que hace viable el flujo moderno de ramas efímeras (feature branches de horas o días) que hoy usan todos los equipos.

> [!info] La intuición original: caminos en el grafo
> Pensá el historial como un mapa de caminos: los commits son ciudades, las ramas son señales que marcan "por aquí va mi trabajo". Avanzás por un camino (commitea en una rama), podés volver atrás, bifurcarte (branch), y **unir caminos** (merge). La diferencia con el mapa real: los caminos de Git son gratuitos y reversibles. Absorbiendo la metáfora clásica del vault: *las ramas son caminos que podemos seguir conforme avanzamos en el repositorio*.

## Anatomía: refs/heads y HEAD

| Concepto | Archivo | Contenido |
|---|---|---|
| Rama | `.git/refs/heads/<nombre>` | SHA del commit al que apunta |
| Rama actual | `.git/HEAD` | `ref: refs/heads/<nombre>` (verificado) |
| Puntero "apegado" | `.git/HEAD` en detached | un SHA directo, sin rama |

```bash
cat .git/HEAD            # ref: refs/heads/main
git symbolic-ref HEAD    # refs/heads/main
git branch -vv           # ramas con upstream y ahead/behind
```

Cuando hacés `commit`, Git mueve el puntero de tu rama al commit nuevo (y `HEAD` lo sigue). Cuando hacés `checkout`/`switch`, movés `HEAD`. Cuando hacés `merge`, creas un commit con dos padres y adelantás la rama actual. **Todo el comportamiento de ramas sale de mover punteros sobre un grafo inmutable** (ver [[Qué es Git]]).

### Ramas remotas: `origin/main` no es "la rama del servidor"

Tras `git fetch`, Git actualiza **refs de solo lectura** `refs/remotes/origin/<rama>` con lo último visto en el remoto:

```
origin/main  →  último SHA que VOS viste de main en el remoto (puede estar desactualizado)
main         →  tu rama local
@{upstream}  →  alias de la rama de tracking (origin/main si la tiene)
```

`origin/main` **no se actualiza sola**: hasta el próximo fetch, Git cree que el remoto está donde lo dejaste. De ahí salen errores aparentemente imposibles ("ya hice pull y me rechaza el push"): alguien avanzó el remoto y tu `origin/main` está vieja. `git fetch -p` la refresca (y con `fetch.prune=true`, borra ramas remotas muertas — ver [[Configuración Inicial]]).

## Crear, entrar y moverse

| Comando | Qué hace |
|---|---|
| `git branch <nombre>` | Crea sin entrar (puntero en HEAD actual) |
| `git branch <nombre> <base>` | Crea desde otra rama/commit |
| `git switch <nombre>` | Cambia a ella (y activa tracking si existe) |
| `git switch -c <nombre>` | **Crea e entra** (la forma moderna) |
| `git switch -c <nombre> <base>` | Crea desde base y entra |
| `git checkout <nombre>` | La forma clásica de cambiar (ver [[Comandos Git]]) |
| `git checkout -b <nombre>` | Clásico "crear e entrar" |
| `git switch -` | Vuelve a la rama anterior |
| `git branch -m <nuevo>` | Renombra la rama actual |
| `git branch -m <viejo> <nuevo>` | Renombra cualquiera |
| `git branch -d <nombre>` | Borra **solo si ya está integrada** (mergeada o fast-forward) |
| `git branch -D <nombre>` | Borra **sin preguntar** — trabajo no integrado queda solo en reflog |
| `git branch -vv`, `git branch -a` | Ramas locales con tracking / todas (locales + remotas) |

`-d` vs `-D` es la red de seguridad: `-d` se niega con `error: The branch 'x' is not fully merged` si el trabajo no se integró — deliberado, porque borrar la rama puede dejar commits accesibles solo vía reflog.

### Detached HEAD: navegar suelto

```bash
git switch --detach v2.1.0    # o git checkout abc1234
```

`HEAD` apunta directo a un commit: los commits nuevos no pertenecen a ninguna rama y quedan "perdidos" (solo reflog) al volver. Es el modo normal de revisar tags o probar un PR sin suciedad. Salida: `git switch main`, o conservar el trabajo: `git switch -c rama-nueva`.

## Merge: los tres escenarios (absorbiendo la taxonomía clásica)

Unir ramas = crear un commit que tiene **dos padres**. La rama actual avanza; la rama fusionada queda como "ya integrada" (mergeada). Hay tres escenarios posibles según el estado del grafo:

### 1. Fast-forward (avance rápido)

La rama destino **no tiene commits propios desde el ancestro común**: no hay nada que unir, Git solo mueve el puntero hacia adelante. **No se crea commit de merge** y no queda registro visible de que hubo una rama (el `--graph` se ve lineal).

```
Antes:  A───B───C  (main)
                 \
                  D───E  (feature)
Después de merge (FF):  A───B───C───D───E  (main y feature apuntan a E)
```

### 2. Merge automático (uniones sin conflicto)

Ambos lados avanzaron, pero tocaron **archivos o líneas distintas**: Git calcula el ancestro común (three-way merge) y fusiona solo. **Sí se crea commit de merge** con dos padres (verificado):

```
*   1a492bc merge resolved      <- commit de merge, 2 padres
|\
| * 6248c97 feat
* | 9a290b6 main work
|/
* 63e7c1d base
```

### 3. Conflicto (fusión manual)

Ambos lados cambiaron **las mismas líneas**: Git no adivina la intención y deja los conflictos marcados en el archivo. Verificado en la práctica:

```bash
$ git merge feature
Auto-merging f.txt
CONFLICT (content): Merge conflict in f.txt
Automatic merge failed; fix conflicts and then commit the result.
```

```
linea1
<<<<<<< HEAD
MAIN
=======
FEATURE
>>>>>>> feature
```

Con `merge.conflictstyle zdiff3` (ver [[Configuración Inicial]]) se añade la sección base `||||||| <sha>` entre los dos lados — se ve la versión original y decidir es mucho más fácil.

### Flujo de resolución completo

```bash
git merge feature              # entra en estado de conflicto
git status                     # muestra "both modified: f.txt" (UU)
# 1. editar f.txt a mano: quitar marcadores, dejar el código correcto
git add f.txt                  # marcar resuelto (puede ser archivo por archivo)
git commit                     # crea el commit de merge (abre editor con "Merge branch 'feature'")
# o desistir:
git merge --abort              # vuelve exactamente al estado previo (verificado)
```

| Comando en conflicto | Efecto |
|---|---|
| `git merge --abort` | Cancela y restaura el estado pre-merge (verificado que funciona y `MERGE_HEAD` desaparece) |
| `git checkout --ours <f>` / `--theirs <f>` | Toma el lado completo de "nuestra rama" / "la otra" (ojo: --ours = HEAD actual, no el merge) y lo pone en el índice y disco |
| `git checkout --conflict=diff3 <f>` | Reformatea los marcadores con sección base |
| `git mergetool` | Editor visual con los tres lados |
| `git diff --name-only --diff-filter=U` | Lista solo los archivos sin resolver |

> [!warning] `--ours` y `--theirs` están invertidos respecto a la intuición
> En un merge, `--ours` es la rama en la que estás parado (HEAD) y `--theirs` la rama que estás fusionando. Pero en un conflicto de **rebase** está al revés: `--ours` = la rama destino (lo que rebasea) y `--theirs` = tu commit original. Por eso conviene leer los marcadores y no confiar en la semántica de los atajos.

Estado interno durante el conflicto: Git escribe `.git/MERGE_HEAD` (SHA de la otra rama), `.git/MERGE_MSG` (mensaje del próximo commit) y marca las etapas 1/2/3 en el índice (`git ls-files -u` muestra base/ours/theirs por zona). `git add` colapsa las tres etapas en la resolución.

### El recurso silencioso: `rerere` y conflicto repetido

`git config rerere.enabled true` (ver [[Configuración Inicial]]) — Git **recuerda cómo resolviste cada conflicto** (por su diff base) y reaplica la resolución si volvés a chocar con el mismo. Imprescindible en flujos con ramas longevas y merges repetidos de la misma rama (`--rebase` con ramas compartidas, bisect con merges). Verificación: `git rerere status`, `git rerere diff`, `git rerere forget <archivo>`.

### Política de merge: ¿fast-forward o no?

| Política | Efecto | Cuándo |
|---|---|---|
| FF (defecto) | Historia lineal si no hay divergencia; sin registro de la rama feature | Equipos que viven en rebase |
| `--no-ff` | **Siempre** commit de merge: se ve "cuándo entró qué feature" | Git-flow, release branches, PRs de GitHub |
| `--ff-only` | Falla si no puede avanzar — prohíbe merges reales | Scripts CI que solo integran linealmente |

```bash
git merge --no-ff feature   # conserva el "nodo" feature en el historial
```

`--no-ff` convierte el historial en documentación viva: `git log --graph` muestra qué commits pertenecieron a cada feature y cuándo se integró. Es la política por defecto de GitHub Flow vía PRs (la plataforma crea el merge commit o el squash).

## Merge vs rebase: la decisión

| | `git merge` | `git rebase` |
|---|---|---|
| Historia | Grafo fiel: conserva la divergencia real | Lineal: reescribe los commits encima |
| Commits nuevos | 1 commit de merge | **Todos** tus commits cambian de SHA |
| Registro | Muestra "aquí hubo dos líneas" | Muestra "siempre fue lineal" |
| Seguro en compartidas | **Sí, siempre** | **No** (reescribe historia publicada) |
| Bisect (`git bisect`) | Puede quedar en commits de merge ambiguos | Historia lineal = búsqueda limpia |
| `git blame` | Salta entre líneas en merges | Lineal, predecible |
| Conflictos | Pueden aparecer una vez, resueltos en el merge | Pueden **repetirse commit por commit** |
| Costo mental | Bajo (un comando) | Alto (repasar el plan) |

Detalle completo del modelo de replay, `--onto` y recuperación en [[Rebase]]. La regla práctica: **tu rama local, rebaseá; la rama compartida, mergeá.**

## Tracking: upstream y ahead/behind

```bash
git push -u origin rama-nueva    # crea el upstream la primera vez
git branch -vv                   # main [origin/main] ahead 2, behind 1
git status -sb                   # ## main...origin/main [ahead 2, behind 1]
git pull                         # usa el upstream configurado
git rev-list --left-right --count HEAD...@{upstream}   # exacto: "2 1"
```

`@{upstream}` es el alias universal de tu rama de tracking. Si `git pull` sin upstream, Git te pide que configures con `--set-upstream` (o `-u` en push). Ver [[Configuración Inicial]] para `pull.rebase` y `push.default`, que definen el comportamiento por defecto de estos comandos.

## Worktrees: dos ramas vivas a la vez

`git worktree` da a un mismo repositorio **múltiples directorios de trabajo**, cada uno con su propio HEAD:

```bash
git worktree add ../hotfix-dir hotfix      # segundo dir con la rama hotfix activa
git worktree list                          # lista los worktrees
git worktree remove ../hotfix-dir          # limpiar
```

Resuelve el problema clásico: no poder hacer `git switch` a main para probar un PR sin arrastrar o stash-ear el trabajo actual (ver [[Stash]]). Un mismo commit vive en dos carpetas sin duplicar el object store (comparten `.git`).

## Flujos de trabajo: GitHub Flow vs git-flow vs trunk-based

| Flujo | Estructura | Ritmo | Para quién |
|---|---|---|---|
| **Trunk-based** | Todo en `main` (rama principal), features como ramas de días + feature flags | Integración diaria, releases continuos | Equipos CD maduros (Google, Facebook) |
| **GitHub Flow** | `main` siempre desplegable + ramas cortas + PR + deploy tras merge | Continuo | La mayoría de productos web hoy |
| **Git-flow** (`main` + `develop` + `release/*` + `hotfix/*`) | Estructura formal de 5 ramas | Versionado periódico con soporte | Software empaquetado con releases programadas |
| **Fork flow** | Nadie pusha al central: fork → rama → PR | Gobernanza estricta | Open source, orgs con control de acceso |

```
GitHub Flow (el estándar 2026):
main ────●────●──────●────●───  (siempre desplegable; PRs merged acá)
           \      / \  /
            ●────●   ●●        (ramas de días, no de meses)
          feature   fix/login
```

Elegir git-flow hoy por costumbre es el error más común de diseño de flujo: su complejidad (`develop`, `release/*`) asume releases empaquetados; para deploys continuos, GitHub Flow o trunk-based son más simples y más seguros.

## Colaboración y seguridad

### La regla de oro de las ramas compartidas

**Nunca hagas force-push a una rama que otros usan.** El push normal ya protege: si el remoto avanzó, Git rechaza con `! [rejected] non-fast-forward`. El peligro es forzar:

| Comando | Riesgo |
|---|---|
| `git push --force` | Sobreescribe lo que sea; si alguien avanzó, **su trabajo desaparece del remoto** |
| `git push --force-with-lease` | Sobreescribe **solo si el remoto está donde tu último fetch lo vio** — la opción responsable |
| `git push --force-if-includes` | Extra: solo si tus commits locales ya están en el remoto (con lease + refspec) |

```bash
git rebase main            # reescribí mi rama local
git push --force-with-lease # empujo la versión reescrita SIN pisar trabajo ajeno
```

`--force-with-lease` es el guardián de `rebase` publicado: si tu compañero empujó mientras tanto, el push falla en vez de borrarle el commit. Detalle del replay en [[Rebase]]. En ramas compartidas, la alternativa correcta a reescribir es `git revert` (ver [[Comandos Git]]).

### Recuperación: rama perdida (verificada)

Borrar una rama borra el puntero, no los commits — el reflog conserva el camino:

```bash
$ git reflog show feature-loss | head -3
52b952a feature-loss@{0}:
e0e3a63 feature-loss@{1}: commit: c3
eca668c feature-loss@{2}: commit: c2-perdido

# repuntar la rama al commit perdido:
git update-ref refs/heads/feature-loss feature-loss@{1}
git log --oneline feature-loss   # c3, c2-perdido, c1 recuperados
```

Flujo humano equivalente (mismo mecanismo, sin tocar refs a mano):

```bash
git reflog                        # localizar el SHA perdido
git branch recuperada <sha>       # crear rama en ese punto
git branch -m perdida recuperada  # o renombrar si el nombre original importaba
```

Verificado de punta a punta: los objetos siguen íntegros y alcanzables hasta que el `git gc` expire el reflog (~90 días por defecto). Igual aplica a commits "perdidos" por `reset --hard` (ver [[Comandos Git]]).

### Buenas prácticas de equipo

1. **Ramas cortas**: días, no semanas — menos divergencia, conflictos más pequeños.
2. **Descriptive names**: `feature/login-oauth`, `fix/bug-123`, `chore/deps`.
3. **Borrá ramas integradas**: `-d` (con su protección) tras el merge; en GitHub el PR merged las borra solas.
4. **Pull antes de push**: `git pull --rebase` para no acumular merges inútiles (ver [[Rebase]]).
5. **Protege `main` en la plataforma**: PRs + checks obligatorios — la defensa real no es costumbre, es política de rama en GitHub/GitLab.

## Errores comunes

1. **`git checkout feature` con trabajo sucio**: Git arrastra cambios si no chocan, y si chocan aborta — pero a veces el arrastre es indeseable. Antes: `git status` limpio o `git stash push -u` (ver [[Stash]]).
2. **Borrar la rama activa**: `git branch -d .` o el nombre actual da `error: Cannot delete branch 'x' checked out at ...`. Cambiate primero (`git switch main`).
3. **Crear rama desde `origin/main` desactualizado**: `git switch -c nueva origin/main` sin fetch = base vieja. Primero `git fetch -p`.
4. **Esperar que `origin/main` se actualice sola tras el push de un compañero**: siempre `git fetch` (o `pull`) primero.
5. **Force-push a main en solitario**: aunque seas el único dev, perdés la red de seguridad del reflog compartido y de los PRs. Usa `--force-with-lease` si acaso, y solo en ramas propias.
6. **Resolviendo conflictos con `--theirs` a ciegas en rebase**: recordar la inversión ours/theirs (arriba) — la resolución manual con marcadores es más segura.

## Cheatsheet

| Tarea | Comando |
|---|---|
| Crear e entrar | `git switch -c <nombre>` |
| Cambiar | `git switch <nombre>` |
| Renombrar | `git branch -m <nuevo>` |
| Ver todas (+ remotas) | `git branch -a` / `git branch -vv` |
| Borrar segura / forzada | `git branch -d` / `-D` |
| Merge visible | `git merge --no-ff <rama>` |
| Abortar merge | `git merge --abort` |
| Resolver conflicto | editar → `git add <f>` → `git commit` |
| Tracking nuevo | `git push -u origin <rama>` |
| Estado vs upstream | `git rev-list --left-right --count HEAD...@{upstream}` |
| Empuje reescrito seguro | `git push --force-with-lease` |
| Rama perdida | `git reflog` → `git branch recuperada <sha>` |
| Segundo directorio | `git worktree add ../ruta <rama>` |

## Relacionadas

- [[Qué es Git]] — el modelo de objetos y refs que hace posible todo esto
- [[Comandos Git]] — comandos básicos, matriz de deshacer y rangos `..`/`...`
- [[Rebase]] — reescribir ramas: replay, `-i`, autosquash y recuperación
- [[Stash]] — cambios sucios antes de cambiar de rama
- [[Configuración Inicial]] — `pull.rebase`, `push.default`, `fetch.prune`, `rerere`
- [[Tags]] — marcar puntos exactos del grafo para releases
- [[Archivos Especiales]] — submódulos como "ramas dentro de ramas" y hooks pre-push
