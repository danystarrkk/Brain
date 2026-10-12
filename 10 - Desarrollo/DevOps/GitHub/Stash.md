---
tags:
  - desarrollo
  - devops
  - stash
tipo: concepto
estado: terminado
---

# Stash

El **stash** es una pila de instantáneas de trabajo sucio que Git guarda aparte para recuperarlas luego: la caja donde dejás los cambios a medio hacer cuando necesitás un working tree limpio **sin inventar un commit de mentira**. Formalmente: `git stash` guarda los cambios del working tree y del índice, y restaura el working tree al estado de `HEAD` — como si no hubieras tocado nada.

> [!info] El stash NO es un backup ni un contenedor mágico
> Es una **pila de commits reales** colgando de una ref especial. Un `git stash` de verdad crea commits (con padres, trees y todo), solo que invisibles al `git log` normal. Ese descubrimiento explica todo: por qué hay `stash@{n}`, por qué se pueden rescatar tras "borrarlos", y por qué **un stash perdido se recupera con la misma técnica que un commit perdido: reflog**.

## Modelo: la pila `refs/stash`

```bash
git stash push -m "wip: formulario"
git stash list
# stash@{0}: On main: wip: formulario     <- el más reciente
# stash@{1}: On main: otra cosa
```

- `stash@{0}` es siempre el último; `{1}` el anterior, etc. — sintaxis del **reflog**, porque el stash vive en la reflog de `refs/stash` (ver [[Comandos Git]] para la sintaxis `HEAD@{n}`).
- Internamente cada stash es un commit con **hasta tres padres** (verificado en la práctica):

| Padre | Contenido |
|---|---|
| `stash^1` | El commit `HEAD` en el momento del stash (base) |
| `stash^2` | Un commit con el estado del **índice** (lo staged) |
| `stash^3` | Un commit con los archivos **untracked** (solo existe con `-u`/`-a`) |

```bash
git rev-parse "stash^!" "stash^1" "stash^2" "stash^3"
# 8253969941...   <- el commit del stash
# ^76206c945...   <- base (HEAD)
# ^e1c9820f2...   <- índice
# ^5a0162773...   <- untracked (creado por -u)
```

Por eso `git stash apply --index` puede restaurar el índice exacto: la información staged se guardó por separado, no fusionada.

> [!warning] El stash es local y frágil
> **Nunca viaja a un remoto**: `git push origin stash` no es un flujo válido (existe un atajo no recomendable con `refs/stash` en el remoto, pero es trampa). No sirve como respaldo ni para compartir trabajo: para eso hay commits y ramas. Y `git gc` puede expurgar stashes huérfanos viejos — si el trabajo importa, **promovelo a commit** (o rama) antes de que madure en la pila.

## Catálogo de banderas de creación

| Bandera | Efecto |
|---|---|
| `git stash` | Atajo de `push` con defectos: guarda **solo archivos trackeados** |
| `-u` / `--include-untracked` | Incluye archivos sin trackear (los pone en el padre 3) — el flag que casi siempre falta |
| `-a` / `--all` | Incluye untracked **e ignorados** (basura incluida — uso raro) |
| `-m "mensaje"` | Descripción legible en `stash list` (reemplaza al obsoleto `stash save`) |
| `-p` / `--patch` | Elige hunk a hunk qué stashear (como `add -p`) |
| `-k` / `--keep-index` | Stashea todo **menos** lo que ya estaba staged — commitear lo listo y guardar lo WIP |
| `--pathspec-from-file <f>` | Stashea solo rutas listadas en un archivo |
| `git stash push -- <ruta>` | Stashea solo esas rutas |
| `--staged` (git ≥ 2.35) | Stashea **solo** lo staged |

```bash
git stash push -u -m "wip: formulario de login"
git stash push -k -m "solo lo WIP"     # staged listo para commit; el resto al stash
git stash push -- src/api/             # solo la carpeta api
```

## Comandos de recuperación y gestión

| Comando | Efecto |
|---|---|
| `git stash list` | La pila, más reciente arriba |
| `git stash apply [stash@{n}]` | Restaura los cambios **sin quitarlo** de la pila (verificado) |
| `git stash pop [stash@{n}]` | Restaura **y elimina** el stash (verificado: `Dropped refs/stash@{0}`) |
| `git stash apply --index stash@{n}` | Restaura también el estado staged exacto |
| `git stash show stash@{n}` | Resumen de archivos y stats del stash |
| `git stash show -p stash@{n}` | El diff completo |
| `git stash branch <nombre> [stash@{n}]` | Crea rama desde el commit base del stash y aplica encima |
| `git stash drop [stash@{n}]` | Borra un stash concreto |
| `git stash clear` | Vacía la pila entera (sin confirmación) |
| `git stash create` / `git stash store` | Modo manual: crea el objeto stash sin aplicarlo (avanzado) |

`apply` vs `pop`: `apply` deja el stash en la pila (por si acaso), `pop` lo descarta. En situaciones con posibilidad de conflicto, la secuencia prudente es `apply` → verificar → `drop` a mano.

Nota sobre la sintaxis histórica: el viejo `git stash save "msg"` está obsoleto — es `git stash push -m "msg"`. Y donde circula `git stash list --stat`, el comando real de stats es `git stash show --stat stash@{n}` (list no acepta stats: la lista es solo refs).

### El pop que conflictúa: verificado

Si cambiaste el archivo después de stashear, `pop` puede conflictuar. Verificado en la práctica — y Git hace lo correcto: **conserva el stash**:

```bash
$ git stash pop
Auto-merging f.txt
CONFLICT (content): Merge conflict in f.txt
The stash entry is kept in case you need it again.
```

Los marcadores del conflicto de stash tienen **etiquetas propias** (no HEAD/rama):

```
<<<<<<< Updated upstream      <- lo que tenés AHORA en el archivo
...
=======
...
>>>>>>> Stashed changes      <- lo que tenía el stash
```

Flujo de resolución: editar → `git add f.txt` → verificar → `git stash drop` cuando el trabajo ya esté seguro (en disco o en un commit). El orden inverso (drop primero) es la forma clásica de perder horas.

### `stash@{n}` y el orden trampa

Aplicar `stash@{1}` **no toca** `stash@{0}` (verificado: la lista queda intacta tras `apply`). Pero tras un `pop` de `{0}`, **todos los índices se renumeran**: lo que era `{2}` pasa a `{1}`. Regla: antes de operar sobre la pila, `git stash list` — nunca asumir posiciones de memoria. Y los mensajes con `-m` son la defensa humana contra la renumeración.

## Flujos verificados

### 1. Conmutación rápida de contexto

```bash
# estoy en medio del formulario, me piden un hotfix:
git stash push -u -m "wip: formulario"   # working tree limpio
git switch main && git switch -c fix/login
# ... fix, commit, merge...
git switch main
git stash pop                             # el wip vuelve tal cual
```

Verificado de punta a punta: tras `push -u`, `git status` queda limpio y los archivos untracked desaparecen del disco del repo; tras `pop`, los cambios tracked y untracked vuelven (untracked como `??` de nuevo).

### 2. Pull con árbol sucio

```bash
git pull               # falla si el pull toca archivos sucios
git pull --autostash   # stashea, integra, reaplica (con pull.rebase.autostash=true es automático)
```

### 3. Checkpoint antes de un rebase/riesgo

```bash
git stash push -u -m "checkpoint antes de rebase -i"
git rebase -i main
# si sale mal además del --abort normal, el checkpoint está ahí
```

Ver [[Rebase]] para la recuperación por reflog y `--abort`.

### 4. Commitear lo staged, guardar el resto (`-k`)

```bash
git add src/api/cliente.ts      # esto ya está listo
git stash push -k -m "lo demás" # el resto va al stash, staged intacto
git commit -m "feat: cliente api"   # commit limpio con lo listo
```

## Trampas y errores comunes

> [!tip] Ciclo de vida del stash en una imagen
>
> `stash push` → commit en la pila (`stash@{0}`) → `apply` (recupera, conserva) → `drop` manual, o `pop` (recupera y descarta), o `branch` (promueve a rama) — y si algo salió mal: `reflog stash` → `apply <sha>`. Todo dentro de ~90 días de expiración del reflog; después, el `git gc` puede expurgarlo.

| Trampa | Qué pasa | Defensa |
|---|---|---|
| **`git stash` sin `-u`** | Los archivos untracked **no se guardan**: quedan en el working tree "limpio" y puedes pisarlos/confundirte | Siempre `-u` (o `-a` consciente) |
| **Stash y luego `clean`** | `git clean -fd` mata los untracked que el stash no guardó | `clean -n` primero; nunca sin `-u` en el stash previo |
| **Conflictos en `pop`** | Si cambiaste el archivo desde el stash, el pop conflictúa y **el stash se conserva** (eso es bueno) | Resolver a mano, `git add`, y `drop` cuando ya no haga falta |
| **Renumeración tras pop/drop** | `stash@{2}` deja de existir como tal | Operar siempre sobre `git stash list` fresco |
| **Stash en equipo** | "¿Por qué mi compañero no ve mi stash?" — es local, siempre | Comunicación + rama/wip commit para compartir |
| **Stash viejo olvidado** | Se acumulan wip de hace meses; el gc puede expurgarlos | `git stash list` periódico; promover o dropear |
| **Stash como respaldo** | Un `git stash clear` y un gc son suficientes para perderlo | Trabajo importante = commit (incluso `wip:` en rama dedicada) |
| **Aplicar en la rama equivocada** | El stash no guarda "de qué rama venía" (solo el mensaje "On main") | Leer el mensaje; preferir `git stash branch <nueva>` que ancla correctamente |

> [!tip] `git stash branch`: el rescatador correcto
> `git stash branch fix-raras stash@{1}` crea la rama **en el commit donde estaba HEAD al hacer el stash** y aplica los cambios encima — se borra el stash solo si el apply sale limpio. Evita el escenario de aplicar un stash en una base más nueva donde conflictúa por accidente de contexto.

## Recuperar un stash borrado

Tras `drop` o `pop`, el mensaje `Dropped refs/stash@{0} (8253969...)` incluye el SHA del commit — y el stash, siendo un commit real, queda en el reflog de `refs/stash`:

```bash
git stash list                    # ya no está
git reflog stash                  # entradas: "drop: On main: wip..." con el SHA
git stash apply 8253969           # resucitar por SHA directo
# o reconstruir la pila:
git stash store -m "rescatado" 8253969
```

Mismo principio que la recuperación de ramas perdidas (ver [[Ramas]]) y del reflog general (ver [[Comandos Git]]): Git borra punteros, no objetos — casi todo es recuperable dentro del periodo de expiración del reflog (~90 días por defecto).

## Alternativas al stash

| Alternativa | Cuándo es mejor |
|---|---|
| **Commit WIP** (`git commit -m "wip"` y luego `reset HEAD^` o `rebase -i` para ocultarlo) | Trabajo largo (días): visible, versionado, con mensaje propio |
| **Worktree** (`git worktree add ../otro fix/login`) | Querés las dos cosas **a la vez**: el sucio en un dir, la otra rama en otro (ver [[Ramas]]) |
| **Rama temporal** (`git switch -c wip-guardada`) | El stash creció demasiado o lo necesitás compartir |
| **`git stash push --pathspec`** para solo algunas rutas | El resto del árbol sigue vivo — mitad de conflicto |

Filosofía: stash para **horas**; commit/rama/worktree para **días**.

Tabla de decisión rápida:

| Situación | Herramienta |
|---|---|
| Cambios de 30 minutos, vuelvo ya | `git stash push -u` |
| Cambios de días, necesito verlos a la vez | `git worktree` (ver [[Ramas]]) |
| Cambios que quiero compartir o revisar | Rama + commit (aunque sea `wip:`) |
| Solo algunas rutas sucias | `git stash push -- <ruta>` |
| Staged listo, resto sin terminar | `git stash push -k` |

## Errores comunes (síntoma → arreglo)

1. **`git stash` dice "No local changes to save"** — tus cambios son untracked y no pusiste `-u`. `git stash push -u`.
2. **`error: your local changes would be overwritten` al hacer pop/apply** — conflictúa con cambios actuales; stash-ear o commitear lo actual primero, y reintentar.
3. **Aparecieron cambios fantasma tras el pop** — son los cambios del stash; `git diff` y `git status` lo confirman. Si no los querías: `git restore .` (con cuidado) o `git checkout -- <f>`.
4. **`git stash clear` sin confirmación** — no lo hay; la pila entera desaparece. Solo con `list` a la vista.
5. **Esperar que el stash llegue al remoto** — no viaja. Para compartir: commit en rama.
6. **Stash aplicado sobre base distinta** — el diff del stash se calculó contra el commit original; aplicar en otra base puede conflictuar "raro". Usar `git stash branch` (arriba).

## Cheatsheet

| Tarea | Comando |
|---|---|
| Guardar (con untracked) | `git stash push -u -m "qué es"` |
| Guardar solo algunas rutas | `git stash push -- src/api/` |
| Guardar solo lo no staged | `git stash push -k` |
| Listar | `git stash list` |
| Recuperar sin borrar | `git stash apply stash@{n}` |
| Recuperar y borrar | `git stash pop` |
| Restaurar también lo staged | `git stash apply --index stash@{n}` |
| Ver el diff | `git stash show -p stash@{n}` |
| Ver stats | `git stash show --stat stash@{n}` |
| Recuperar en rama nueva | `git stash branch <nombre> stash@{n}` |
| Borrar uno / todos | `git stash drop` / `git stash clear` |
| Resucitar borrado | `git reflog stash` → `git stash apply <sha>` |
| Autostash en pull/rebase | `git pull --autostash` / config `rebase.autoStash` |

## Relacionadas

- [[Comandos Git]] — la matriz de deshacer y el reflog sobre el que se apoya todo el stash
- [[Ramas]] — worktrees y cambio de rama con trabajo sucio: los rivales del stash
- [[Rebase]] — checkpoints antes de replantear historia y `autoStash`
- [[Qué es Git]] — por qué un stash son commits reales con tres padres
- [[Configuración Inicial]] — `pull.rebase.autostash` y `rebase.autoStash`
