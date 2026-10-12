---
tags:
  - desarrollo
  - devops
  - tags
tipo: concepto
estado: terminado
---

# Tags

Un **tag** (etiqueta) es una referencia que marca un commit exacto — normalmente una versión (`v2.1.0`), un release o un punto histórico. A diferencia de las ramas, los tags **no se mueven**: son anclas inmutables del historial. Formalmente son refs en `.git/refs/tags/<nombre>`, pero con un detalle de diseño importante: existen dos tipos, y solo uno crea un objeto propio.

> [!info] ¿Por qué existen si ya existen las ramas?
> Las ramas apuntan a "dónde va el trabajo"; los tags apuntan a "dónde estuvo". `main` avanza; `v2.1.0` nunca. Y el tipo anotado añade lo que una rama no puede tener: **metadatos firmables** (quién etiquetó, cuándo, con qué mensaje) — la base de la trazabilidad de releases.

## Lightweight vs anotado: el modelo de objetos verificado

| | Tag ligero (*lightweight*) | Tag anotado (*annotated*) |
|---|---|---|
| Qué es | Un archivo con un SHA — igualito que una rama | Un **objeto `tag`** propio: tagger, fecha, mensaje, puntero, firma opcional |
| `git cat-file -t` | `commit` (apunta directo al commit) | `tag` (objeto intermedio) |
| SHA del tag | = SHA del commit | **Distinto** al del commit |
| Mensaje/fecha de etiquetado | No existe | Sí (`git tag -n`, `git show`) |
| Firma GPG/SSH | No | Sí (`-s`, `git tag -v`) |
| Recomendado para | Uso interno, atajos personales | **Releases y todo lo que consuma otra persona** |

Verificado en la práctica — la diferencia de objeto es real:

```bash
git tag -a v1.0 -m "release 1.0"
git tag v1.0l                          # ligero apuntando al mismo commit

git cat-file -t v1.0                  # tag        <- objeto propio
git cat-file -t v1.0l                 # commit     <- apunta directo al commit

git rev-parse v1.0l                    # e06cbcdd...  (= SHA del commit)
git rev-parse v1.0                     # 597f800a...  (SHA del objeto tag)
git rev-parse v1.0^{commit}            # e06cbcdd...  (el commit que señala)
```

`v1.0^{commit}` (pronuncie "peel el tag") es el recurso para "el commit detrás de cualquier cosa": funciona con tags, ramas y commits — ver [[Qué es Git]] para el grafo de objetos que hace falta para entenderlo.

> [!warning] El costo de olvidarse del `-a`
> Un tag ligero no tiene autor ni mensaje: `git show v1.0` muestra el commit, no "quién y por qué lo marqué". Si el tag es un release público o algo que auditará seguridad, **siempre `-a` (mejor `-s`)**. Distinguirlos al revés: `git for-each-ref --format='%(objecttype) %(refname:short)' refs/tags`.

## Crear tags

| Comando | Qué hace |
|---|---|
| `git tag <nombre>` | Ligero en el último commit |
| `git tag <nombre> <commit>` | Ligero en cualquier commit (`v0.9 a1b2c3d`) |
| `git tag -a <nombre> -m "mensaje"` | Anotado en HEAD |
| `git tag -a <nombre> <commit> -m "mensaje"` | Anotado en un commit específico |
| `git tag -s <nombre> -m "mensaje"` | Anotado **firmado** (requiere GPG/SSH — ver [[Configuración Inicial]]) |
| `git tag -f <nombre> <commit>` | **Mueve** un tag existente (solo si nadie lo consumió; requiere force en push) |
| `git tag -a <nombre> <commit> --cleanup=verbatim` | Conserva el mensaje literal (por defecto limpia comentarios) |

```bash
git tag -a v2.1.0 -m "Release 2.1.0: API de pagos + fix de sesiones"
git tag -s v2.1.0 -m "v2.1.0 firmado"    # + tag.gpgsign=true para automático
```

Con `tag.gpgsign = true` (ver [[Configuración Inicial]]) todo `git tag -a` sale firmado sin `-s`.

## Listar, buscar y ordenar

| Comando | Qué hace |
|---|---|
| `git tag` | Lista todos (orden alfabético por defecto) |
| `git tag -l "v2.*"` | Patrón glob |
| `git tag -n1` | Lista con la primera línea del mensaje (anotados) |
| `git tag --sort=version:refname` | **Orden semántico** — el correcto |
| `git tag --sort=-version:refname` | Descendente (más nuevo primero) |
| `git tag --sort=creatordate` | Por fecha de creación del objeto tag |
| `git tag --contains <sha>` | Tags que contienen ese commit (¿en qué release entró este fix?) |
| `git tag --merged main` | Tags ya integrados en main |
| `git for-each-ref refs/tags --format='%(refname:short) %(creatordate:short)'` | Vista completa |

> [!warning] La trampa del orden alfabético (verificada)
> El orden por defecto es lexicográfico: `v0.1.0`, `v0.10.0`, `v0.2.0` — `0.10` aparece **antes** que `0.2`. `git tag --sort=version:refname` lo corrige (`v0.1.0`, `v0.2.0`, `v0.10.0`). Scripts que tomen "el último tag" con `tail` sobre la lista cruda eligen mal — siempre `--sort=version:refname`, y mejor `git describe`.

## Ver, eliminar y compartir

| Comando | Qué hace |
|---|---|
| `git show <tag>` | Tag completo: mensaje, tagger, y diff del commit |
| `git show <tag>^{} ` | Solo el commit de abajo (peel) |
| `git tag -d <tag>` | Borra el tag **local** |
| `git push origin <tag>` | Publica **un** tag |
| `git push origin --tags` | Publica **todos** los tags locales |
| `git push --follow-tags` | Publica los anotados alcanzables desde lo que empujás |
| `git push origin :refs/tags/<tag>` | Borra el tag **del remoto** |

> [!warning] Los tags NO se empujan con la rama
> `git push` solo sube refs de ramas. Tras `git tag -a v2.1.0` y `git push`, el tag **no** está en el remoto — el release de GitHub/CI no lo ve. Solución explícita: `git push origin v2.1.0`, o `--follow-tags` en el push, o `push.followTags=true` en la config. Es el bug de release más frecuente de todos los tiempos.

Borrar tags del remoto tiene política: si alguien ya los usó (CI, dependencias fijadas), moverlos rompe referencias — igual que reescribir historia compartida (ver [[Rebase]]). La defensa en plataformas es proteger tags (`protected tags` en GitHub).

## Verificar firmas

```bash
git tag -v v2.1.0              # verifica la firma del tag anotado
git verify-tag v2.1.0
git verify-commit <sha>        # nivel commit
git log --show-signature       # verifica toda la cadena
```

Con SSH como formato de firma (moderno, sin servidores de claves): `gpg.format = ssh` + `user.signingkey` + `gpg.ssh.allowedSignersFile` — ver [[Configuración Inicial]]. El valor para una cadena de suministro: un tag firmado prueba que el binario que bajó un usuario salió del maintainer y no fue tocado en tránsito.

## Versionado semántico (SemVer)

Convención dominante para nombrar releases — `MAJOR.MINOR.PATCH`:

| Componente | Cuándo sube | Ejemplo |
|---|---|---|
| **MAJOR** | Cambios incompatibles con versiones previas (breaking changes) | `v1.4.2 → v2.0.0` al romper la API |
| **MINOR** | Funcionalidad nueva, compatible | `v1.4.2 → v1.5.0` |
| **PATCH** | Correcciones de bugs compatibles | `v1.4.2 → v1.4.3` |

Pre-release y build (reglas de precedencia SemVer 2.0.0):

| Forma | Significado | Orden (de menor a mayor) |
|---|---|---|
| `v1.0.0-rc.1`, `-beta.2`, `-alpha` | Pre-release: **no** es estable | `alpha < beta < rc < release` |
| `v1.0.0+build.5` | Build metadata: no afecta al orden | igual a `v1.0.0` |
| `v1.0.0-rc.1 < v1.0.0` | El pre-release ordena **antes** del release | clave en CI |

Advertencia práctica con tags pre-release en ecosistemas con paquetes: `npm` trata `-rc.1` como pre-release (no lo instala `npm install pkg` sin flag); Debian/RPM tienen reglas propias para `~` y `+`. Y recordar que el prefijo `v` es convención (`v2.1.0`), aceptada universalmente aunque SemVer diga `2.1.0`.

Convenciones de nombres comunes:

| Patrón | Uso |
|---|---|
| `v2.1.0` | SemVer — el estándar |
| `release-2026-03-15` | Versionado por fecha (SaaS continuo) |
| `v2.1.0-rc.1` | Release candidate antes del release |
| `snapshot-main` / `nightly` | Tags movibles de builds automáticos (¡la única excusa para `-f`!) |

## Checkout a tag: el HEAD colgado

```bash
git switch --detach v2.1.0    # o git checkout v2.1.0 (te deja en detached HEAD)
```

Ver [[Ramas]] para detached HEAD: los commits nuevos no pertenecen a ninguna rama. Flujo seguro para probar un release:

```bash
git switch --detach v2.1.0
npm ci && npm test            # probar exactamente esa versión
git switch main               # volver
```

Para arreglar algo de esa versión: `git switch -c hotfix/v2.1.1 v2.1.0` — crea la rama del fix desde el tag (ramas de mantenimiento por release).

## `git describe`: identificación automática de builds

```bash
git describe --tags                    # v0.1.0-2-g81b5d2c
git describe --tags --abbrev=0         # v0.1.0
git describe --always                  # 422ddd6   (fallback sin tags cercanos)
```

Verificado: el formato es `<tag>-<commits desde el tag>-g<sha corto>`. `v0.1.0-2-g81b5d2c` = "2 commits después de v0.1.0, en el commit 81b5d2c". Es la identificación estándar de binarios: inyectarla en el artefacto (`--version` de un CLI, header de un container) permite saber **exactamente** de qué punto del historial vino cada build — sin tags, `--always` da el hash crudo.

## Tags en CI/CD

El disparador estándar: pushes de tags que matchean un patrón:

```yaml
# GitHub Actions (esquema)
on:
  push:
    tags: ['v*']
```

Patrones de pipeline típicos: al ver tag `v*` → build → tests → `gh release create "$TAG" --generate-notes` → publicar artefacto con `git describe`. Pre-release en SemVer → publicar como prerelease. Cadenas de suministro: verificar la firma del tag antes de publicar (`git verify-tag`). Release automation nativa de GitHub: `gh release create v2.1.0` crea el release y adjunta binarios desde la CLI — ver sección siguiente.

### Releases en la plataforma (GH CLI)

```bash
gh release create v2.1.0 --title "v2.1.0" --generate-notes   # release con notas auto
gh release create v2.1.0 ./dist/app.tar.gz                   # con binarios adjuntos
gh release list
```

`gh` es la CLI de GitHub (la plataforma, distinta de Git — ver [[Qué es Git]]). El flujo moderno: tag anotado firmado + `--follow-tags` + `gh release create` = release trazable de principio a fin.

## Recetas sobre releases

### Changelog entre dos versiones

```bash
git log --oneline v2.0.0..v2.1.0                    # todo lo que entró en la release
git log --oneline v2.0.0..v2.1.0 --no-merges        # sin commits de merge
git log --oneline v2.0.0..HEAD --grep="^feat"       # solo features pendientes
```

Con `git describe` identificando cada build (arriba), un changelog automático es un `git log` entre tags consecutivos ordenados con `--sort=version:refname`.

### Hotfix desde un tag viejo

```bash
git switch -c hotfix/v2.1.1 v2.1.0    # rama de mantenimiento desde el release
git cherry-pick <sha-del-fix>          # traer el fix de main (ver Comandos Git)
git tag -a v2.1.1 -m "hotfix" && git push --follow-tags
```

El flujo de ramas de mantenimiento por versión es el único escenario donde git-flow tenía razón — ver la comparación de flujos en [[Ramas]].

### ¿Qué tag incluye mi fix?

```bash
git tag --contains <sha>               # primer release que lo incluye
git tag --contains <sha> --sort=version:refname | head -1
```

Imprescindible en respuestas de seguridad: "¿está afectada la versión desplegada X por el fix Y?" se responde con este comando.

## Errores comunes

1. **Tag ligero en un release público** — sin autor ni mensaje: imposible auditar. Usar `-a` o `-s`.
2. **Olvidar `--tags`/`--follow-tags` al pushear** — el remoto (y el CI) no ven el release. `git push --follow-tags`.
3. **Asumir orden lexicográfico** en "el último tag": `v0.10` antes que `v0.2`. `--sort=version:refname` o `git describe`.
4. **Mover tags publicados** (`-f` + force push de refs/tags) — rompe builds y dependencias fijadas. Si ya pasó: publicar el siguiente número de versión, no reparar el viejo.
5. **Tags en remoto que nadie borra** — la lista crece sin control; política: tags de release = inmutables y protegidos; tags desechables (nightly) en un namespace aparte (`nightly-2026-10-11`).
6. **Esperar que el tag siga al código tras un rebase** — los tags apuntan a SHAs; si reescribís historia, los viejos tags quedan colgados de commits huérfanos. Después de un rebase importante, revisar `git tag --merged` (ver [[Rebase]]).
7. **Confundir tag con branch en scripts** — `git checkout main` vs `git checkout v2.1.0` deja en modo distinto (tracking vs detached).

## Cheatsheet

| Tarea | Comando |
|---|---|
| Tag anotado de release | `git tag -a v2.1.0 -m "v2.1.0"` |
| Tag firmado | `git tag -s v2.1.0 -m "v2.1.0"` |
| En un commit viejo | `git tag -a v2.1.0 <sha> -m "..."` |
| Lista ordenada | `git tag --sort=version:refname` |
| Ver | `git show v2.1.0` / `git tag -n1` |
| Verificar firma | `git tag -v v2.1.0` |
| Publicar | `git push origin v2.1.0` o `git push --follow-tags` |
| Borrar local / remoto | `git tag -d v2.1.0` / `git push origin :refs/tags/v2.1.0` |
| Probar una versión | `git switch --detach v2.1.0` |
| Identificar build | `git describe --tags --always` |
| Release con GH CLI | `gh release create v2.1.0 --generate-notes` |
| Commit detrás de un tag | `git rev-parse v2.1.0^{commit}` |

## Relacionadas

- [[Qué es Git]] — el objeto `tag` en el modelo de cuatro tipos y las refs
- [[Comandos Git]] — `git show`, rangos y la inspección sobre la que se apoyan los tags
- [[Ramas]] — detached HEAD al probar tags y ramas de mantenimiento desde un tag
- [[Rebase]] — qué le pasa a un historial etiquetado cuando se reescribe
- [[Configuración Inicial]] — firmado (GPG/SSH), `tag.gpgsign`, `push.followTags`
