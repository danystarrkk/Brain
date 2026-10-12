---
tags:
  - desarrollo
  - devops
  - rebase
tipo: concepto
estado: terminado
---

# Rebase

**Rebase** mueve (o replica) tus commits para que queden encima de otro punto del historial, transformando una historia divergente en una lineal. Es la herramienta más útil y más peligrosa de Git: en tu rama privada produce historiales impecables; sobre una rama compartida destruye el trabajo de otros. Un rebase bien hecho cubre las capacidades clásicas: **ordenar commits, corregir mensajes, unir (squash) commits y separarlos** — todo vía `rebase -i`.

> [!info] La regla que resume la nota entera
> **Rebase reescribe historia: los commits que mueve nacen de nuevo con SHA nuevo.** Si los viejos estaban publicados y alguien los usó, el rebase crea una historia paralela incompatible. Por eso: *rebaseá lo que no compartís; mergeá lo que sí*.

## Mecánica: qué hace Git al rebasar

Con `main` avanzando y tu rama colgada del commit viejo:

```
Antes:
main:    A───B───C───D          (C y D son de tus compañeros)
              \
tu-rama:      E───F             (E y F tuyos, colgadas de B)

git switch tu-rama && git rebase main
Después:
main:    A───B───C───D
                        \
tu-rama (nuevos E', F'): E'───F'
```

Pasos internos (verificado mentalmente contra el modelo de objetos de [[Qué es Git]]):

1. Git encuentra el **ancestro común** de `tu-rama` y `main` (aquí `B`).
2. Calcula el *patch series* de `E` y `F` respecto de `B` (la diferencia real).
3. **Reaplica cada commit uno a uno sobre `D`** — es un cherry-pick iterativo: cada commit se recrea como objeto nuevo (`E'`, `F'`) con el mismo parche, mensaje y autoría, pero **SHA distinto** (porque el padre y el committer cambiaron).
4. Repunta `tu-rama` a `F'`. Los viejos `E` y `F` quedan huérfanos (el reflog los conserva).
5. Si un parche no aplica limpio, Git detiene el rebase en ese commit y pide resolver — como un merge repetido.

> [!tip] Author vs committer vuelve a aparecer
> En `E'`, el **author** sigue siendo vos (la autoría se respeta — ver [[Qué es Git]]), pero el **committer** pasa a ser quien ejecuta el rebase con fecha nueva. `git log --format="%h %an %cn"` lo muestra. Detalle clave para entender por qué "todos mis commits aparecieron con fecha de hoy" tras un rebase.

## Rebase interactivo: `git rebase -i`

```bash
git rebase -i HEAD~4          # rework de los últimos 4 commits
git rebase -i main            # todo lo que mi rama tiene sobre main
git rebase -i --autosquash main   # ver Autosquash abajo
```

Git abre un editor con el plan de replay — una línea por commit, en orden (del más viejo al más nuevo):

```
pick e5a91c2 feat: añadir endpoint de login
pick 9f30b2a wip
pick 44c8d01 fix typo
pick 1b7a5aa feat: añadir logout
# Rebase 3d9f1ab..1b7a5aa onto 3d9f1ab (4 comandos)
```

### Acciones disponibles (catálogo completo)

| Acción | Efecto | Equivalente |
|---|---|---|
| `pick p` | Mantiene el commit tal cual (reaplica) | — |
| `reword r` | Reaplica y abre el editor para cambiar **solo el mensaje** | `commit --amend -m` |
| `edit e` | Para en ese commit: podés amend, separar, añadir | pausa del rebase |
| `fixup f` | Fusiona con el anterior **descartando este mensaje** | `commit --fixup <sha>` |
| `squash s` | Fusiona con el anterior **abriendo editor para combinar mensajes** | `commit --squash <sha>` |
| `drop d` | Elimina el commit | — |
| Reordenar líneas | Cambia el orden de replay | reordenar historia |
| `break` | Pausa aquí sin hacer nada (inspeccionar, `git rebase --continue`) | — |
| `exec <cmd>` | Ejecuta un comando después de cada paso (`exec npm test`) | verificación automática |
| `label <n>` | Marca un punto del replay (`label onto`) | para merges |
| `reset <n>` | Vuelve HEAD a un label (descarta lo replayeado desde ahí) | base para replantear |
| `merge <t> <m>` | Crea un commit de merge en el replay (con `--rebase-merges`) | preservar merges |
| `update-ref <ref>` | Actualiza una rama a un punto del replay | conservar referencias |

`edit` es la más potente: pausa el rebase **en** el commit, y desde ahí podés:

```bash
git commit --amend -m "mensaje nuevo"     # reword
git reset HEAD^                            # "descomponer" el commit (cambios quedan en disco)
git add -p && git commit -m "parte 1"     # y volver a partirlo en varios
git rebase --continue
```

`exec npm test` convierte al rebase en un **verificador**: Git corre el comando tras cada replay y se detiene si falla, dejando la rama en el commit problemático.

### Semántica del rango: `^` vs `~`

| Forma | Significado |
|---|---|
| `HEAD~3` | Tres padres hacia **atrás por primera generación** (salta merges) |
| `HEAD~2..HEAD` | Los últimos 2 commits |
| `HEAD^` | El **padre** de HEAD (en merges: primer padre) |
| `HEAD^2` | Segundo padre (el lado fusionado en un merge) |
| `A..B` | Rango abierto: commits en B no en A |
| `A...B` | Diferencia simétrica (ver [[Comandos Git]]) |
| `--root` | Reescribe desde el commit raíz (todo el historial) |

```bash
git rebase -i HEAD~3     # exactamente los 3 últimos
git rebase -i main       # "todo lo que difiero de main" — el uso diario
git rebase -i --root     # el historial completo (¡solo si nunca se publicó!)
```

Con `rebase.instructionFormat = %h %s` (ver [[Configuración Inicial]]) el plan muestra los mensajes reales en cada línea — evita `pick` a ciegas.

### Flags de replay

| Flag | Efecto |
|---|---|
| `--rebase-merges` | Preserva la estructura de merges del rango (en vez de aplanar todo) |
| `--fork-point` | Usa el historial del reflog para detectar el punto de bifurcación real (útil con ramas locales sucias) |
| `--empty=keep\|drop` | Qué hacer con commits que quedaron vacíos tras el replay |
| `--exec <cmd>` | Atajo de `exec` en el plan |
| `--interactive` / `-i` | Plan editable (implícito en los ejemplos) |

`--rebase-merges` produce grafos no lineales "recolocados" — útil para mantener merges de integración grandes sin aplanarlos.

## Autosquash: el workflow profesional de limpieza

La secuencia de "commits de prueba" (`wip`, `fix typo`, `fixup`) se limpia **automáticamente**:

```bash
git commit --fixup e5a91c2      # mensaje: "fixup! feat: añadir endpoint de login"
git commit --fixup e5a91c2      # otro arreglo al mismo commit
git commit --squash 9f30b2a     # "squash! wip" — propondrá mensaje combinado
git rebase -i --autosquash main
# el editor YA viene armado:
# pick   e5a91c2 feat: añadir endpoint de login
# fixup  a1b2c3d fixup! feat: añadir endpoint de login
# fixup  d4e5f6a fixup! feat: añadir endpoint de login
# squash 7788f90 squash! wip
```

Flujo recomendado: trabajá con commits sucios y `--fixup` sin miedo; la limpieza queda para antes del push. Automatizable con alias:

```bash
git config --global alias.fixup '!f() { git commit --fixup=$1 && git rebase -i --autosquash $1^; }; f'
```

Para ciertas ramas se puede automatizar el rebase al pushear:

```bash
git config rerere.enabled true              # recordar resoluciones (ver [[Ramas]])
git config rebase.autoStash true            # stash automático si hay trabajo sucio
git config branch.<rama>.rebase true        # solo esa rama usa rebase al pull
```

## Rebase upstream: `pull --rebase` vs merge

`git pull` integra lo remoto en tu rama. Dos filosofías:

| | `git pull` (merge, defecto) | `git pull --rebase` |
|---|---|---|
| Resultado | Commit de merge ("Merge branch main") | Historia lineal: tu trabajo re-colocado encima de lo nuevo |
| Conflictos | Se resuelven una vez, en el merge | Pueden repetirse por cada commit del replay |
| Historia | Muestra la bifurcación | Muestra línea recta |
| Costo mental | Mínimo | Mayor si divergiste mucho |

```bash
git pull --rebase                 # esta vez
git config pull.rebase true       # para siempre (ver [[Configuración Inicial]])
git pull --rebase --autostash     # aunque tengas archivos sucios
```

La combinación `fetch.prune=true` + `pull.rebase=true` + `rerere.enabled` es el setup de historial limpio para equipos que viven en ramas cortas.

## `git rebase --onto`: injertar ramas

`--onto <nuevo-base> <viejo-base> <rama>` responde: *"mírame los commits de `<rama>` que NO estaban en `<viejo-base>` y replantéalos sobre `<nuevo-base>`"*. El caso clásico: `ramaB` avanzó y vos seguís trabajando en `ramaA` que nació de `ramaB` — querés que `ramaA` cuelgue de `main` sin arrastrar los commits de `ramaB`:

```
Antes:
main:    A
              \
ramaB:       B1───B2          (los B no los quiero en ramaA)
                 \
ramaA:           A1───A2       (lo que sí quiero publicar)

git rebase --onto main ramaB ramaA
Después:
main:    A───A1'───A2'          (ramaA ahora colgada de main, sin B1/B2)
```

```bash
git rebase --onto main ramaB ramaA   # "saca de ramaA lo que no está en ramaB y póntelo sobre main"
git branch -f ramaA ramaB-movida     # variantes con update-ref para fines parecidos
```

Anatomía del comando: **`--onto main`** = destino; **`ramaB`** = corte trasero (excluido, todo lo anterior se descarta); **`ramaA`** = rama a mover. Errores típicos: invertir destino y origen, o usar `ramaB` cuando hay que usar `ramaB~2` (el commit exacto tras el cual todo lo tuyo empezó). Verificación segura antes de mover punteros:

```bash
git log --oneline main..ramaA        # ¿qué tiene ramaA que main no? (antes)
git range-diff main..ramaA~2 main..ramaA  # compara las dos versiones de la serie
git range-diff v1..v2                # la herramienta de auditoría post-rebase
```

`git range-diff` existe precisamente para auditar rebases: compara par a par los parches viejos y nuevos, confirmando que el replay no alteró contenido.

## Rebasing ramas publicadas: la zona de peligro

```bash
git rebase main          # reescribí mi rama...
git push                 # rechazado: el remoto tiene los commits viejos
git push --force         # ¡peligro! puede borrar trabajo ajeno
git push --force-with-lease   # seguro: falla si alguien avanzó desde mi último fetch
```

`--force-with-lease` compara tu `refs/remotes/origin/<rama>` con el remoto real: si difieren (alguien empujó), aborta. Es el único force-push aceptable en rama propia verificable. En ramas **compartidas** (main, develop, cualquier rama de más de una persona): la respuesta correcta a "ya publiqué y quiero reorganizar" es `git revert` de los commits (ver [[Comandos Git]]) — historia aditiva, nunca reescritura.

> [!warning] El accidente clásico: rebasear main por error
> `git rebase main` estando **en main** con main desactualizado, o `git pull --rebase` mal configurado sobre una rama compartida, genera historia divergente que el siguiente push rechaza o, con force, rompe para todos. El seguro: `git config push.default simple` (defecto) + nunca `--force` en compartidas + `main` protegida en la plataforma (ver [[Ramas]]).

## Recuperarse de un rebase mal hecho

El reflog es el salvavidas — todo rebase deja rastro (verificado el mecanismo de reflog en [[Comandos Git]]):

```bash
git rebase main             # rebase que no quería
git reflog                  # entrada: "rebase (finish): ... onto main"
# HEAD@{1} es donde estaba antes del rebase
git reset --hard HEAD@{1}   # o bien:
git rebase --abort          # si el rebase está EN curso (conflicto sin resolver)
```

Dos escenarios distintos:

| Situación | Recuperación |
|---|---|
| Rebase **en curso** (conflicto o pausa) | `git rebase --abort` (vuelve exacto al estado previo) o `--skip`, `--continue` |
| Rebase **terminado** y me arrepentí | `git reflog` → identificar `HEAD@{n}` previo → `git reset --hard HEAD@{n}` |
| Rebase terminado **y ya empujado** | Deshacer local + force-with-lease, o `git revert` si otros lo consumieron |

`ORIG_HEAD` apunta al commit desde el cual arrancó la última operación que puede mover punteros (rebase, reset, merge): `git reset --hard ORIG_HEAD` es el atajo de una línea tras un rebase terminado. Los commits viejos siguen en el object store hasta el GC: si el `reset` no basta, `git reflog` encuentra los SHAs originales para repuntar con `git branch` (verificado en [[Ramas]]).

## Rebase vs merge: comparación completa

| Criterio | Merge | Rebase |
|---|---|---|
| Integridad histórica | Mantiene el registro real de la divergencia | Presenta linealidad reconstruida |
| SHA de los commits | Estables | **Cambian** (authoría sí se conserva) |
| Seguridad en compartidas | Total | Ninguna sin force |
| `git log --oneline` | Mezcla ramas + merge commits | Línea recta, legible |
| `git bisect` | Puede caer en commits de merge poco diagnósticos | Búsqueda binaria limpia |
| `git blame` | Puede saltar por merges | Directo |
| `git revert` de una feature | Difícil (hay que revertir el merge o cada commit) | Cada commit es revertible individualmente |
| Conflictos | Una resolución | Potencialmente una por commit |
| Curva de aprendizaje | Nula | Alta (`-i`, `--onto`, recuperación) |

Síntesis: merge documenta lo que pasó; rebase documenta lo que "debería haber pasado". Ambos legítimos — el error es usar cada uno en el contexto del otro.

## Cuándo NO rebasar

1. **Ramas compartidas** — jamás. Destruye el trabajo sincronizado de otros (ver aviso).
2. **Commits ya consumidos** — si alguien hizo `checkout <sha>` o los referencia (issues, PRs, bisect), los SHA nuevos rompen referencias.
3. **Historia de más de ~10 commits publicados** — el coste del force-push supera el beneficio estético.
4. **Equipos sin acuerdo explícito** — un historial reescrito "por gusto" de un integrante es una bomba de conflictos para los demás.
5. **Merge commits complejos que querés preservar** (sin `--rebase-merges`): aplanarlos borra información de integración que luego necesitás para `git log --graph --merges`.
6. **Antes de entender la recuperación** — si no sabés usar `git reflog` y `--abort`, no arranques un `-i` sobre trabajo valioso.

## Errores comunes

| Síntoma | Causa | Arreglo |
|---|---|---|
| `error: could not apply <sha>...` a mitad del rebase | El parche no aplica (conflicto repetido) | Resolver, `git add`, `git rebase --continue` (o `--skip` para saltar ese commit) |
| `hint: ... use 'git rebase --skip'` | El commit quedó vacío tras el replay | `--skip` |
| Historia duplicada tras push | Rebase de rama publicada sin lease | `git push --force-with-lease` (o revertir el rebase desde reflog) |
| Commits de otra persona con mi fecha | Normal: rebase conserva author, cambia committer | Sin arreglo necesario (es comportamiento) |
| `rebase -i` abrió el editor con el plan equivocado | Rango mal calculado | `git rebase --abort`, reintentar con el rango correcto (`main`, `HEAD~n`) |
| Rebase continuo conflictivo con `pull` | `pull.rebase=true` + ramas muy divergentes | Volver a merge puntual (`git pull --no-rebase`) y converger |

## Cheatsheet

| Tarea | Comando |
|---|---|
| Poner mi rama al día (lineal) | `git fetch && git rebase main` |
| Pull con rebase | `git pull --rebase` (o config `pull.rebase=true`) |
| Reorganizar últimos n | `git rebase -i HEAD~n` |
| Arreglar mensaje | `git rebase -i HEAD~1` → `reword` |
| Unir commits | `-i` → `fixup`/`squash`, o `--fixup` + `--autosquash` |
| Borrar commits | `-i` → `drop` |
| Pausar para edit complejo | `-i` → `edit` |
| Injertar sobre otra base | `git rebase --onto <destino> <corte> <rama>` |
| Verificar que aplicó bien | `git range-diff <antes> <después>` |
| Abortar (en curso) | `git rebase --abort` |
| Deshacer (terminado) | `git reflog` → `git reset --hard HEAD@{n}` |
| Push de rama reescrita | `git push --force-with-lease` |

## Relacionadas

- [[Ramas]] — el grafo sobre el que opera el replay y la política merge-vs-rebase
- [[Comandos Git]] — rangos, reflog y la matriz de deshacer que sostiene la recuperación
- [[Qué es Git]] — por qué los commits "nacen de nuevo" (objetos + SHA por contenido)
- [[Configuración Inicial]] — `pull.rebase`, `rebase.autoStash`, `rerere`, `instructionFormat`
- [[Stash]] — `--autostash` y stash manual antes de replantear historia
- [[Tags]] — cómo afecta reescribir historia a los tags existentes (no los mueve solo)
