---
tags:
  - desarrollo
  - devops
  - inicial
tipo: concepto
estado: terminado
---

# Configuración Inicial

Toda instalación de Git necesita al menos tres cosas antes del primer commit: **identidad** (`user.name` / `user.email`), **rama por defecto** (`init.defaultBranch`) y — si vas a colaborar — un **mecanismo de autenticación** contra los remotos (credenciales HTTPS o llave SSH). Todo lo demás son optimizaciones que vale la pena hacer una vez y olvidar.

> [!info] Git es un programa con configuración en cascada
> Git lee configuración de varios archivos **en orden de prioridad** y la fusiona: lo último gana. Entender esa cascada explica por qué "cambié la config y no pasó nada" (la escribiste en el scope equivocado) y cómo tener valores distintos por equipo, por máquina y por repositorio.

## Scopes de configuración: la cascada completa

| Scope | Archivo típico | Alcance | Cuándo usarlo |
|---|---|---|---|
| **system** | `/etc/gitconfig` (Linux), `C:\Program Files\Git\etc\gitconfig` (Windows) | Todos los usuarios de la máquina | Admin de sistemas, políticas de empresa |
| **global** | `~/.gitconfig` o `~/.config/git/config` | Todos tus repos | Tu identidad, alias, defaults personales |
| **local** | `<repo>/.git/config` | Solo ese repositorio | Config del proyecto (remote, settings de equipo) |
| **worktree** | `<repo>/.git/worktrees/<wt>/config.git` | Ese worktree específico | Exótico: config distinta por worktree (ver [[Ramas]]) |

**Prioridad**: `system` < `global` < `local` < `worktree`. Es decir, un valor local pisa al global, y el global pisa al system.

```bash
git config --list --show-origin        # toda la config efectiva + de qué archivo sale cada clave
git config --get user.email             # valor efectivo (resolviendo la cascada)
git config --global user.email "tu@correo.com"
git config --local core.autocrlf input  # solo este repo, pisa al global
git config --show-origin --get-regexp '^user\.'   # ¿de dónde viene user.*?
```

```ini
# ~/.gitconfig — equivalente directo de los flags --global
[user]
    name = Tu Nombre
    email = tu@correo.com
[init]
    defaultBranch = main
```

```bash
git config --global -e   # abre ~/.gitconfig en tu editor (core.editor) — la vía cómoda de editar
git config --local -e    # abre la config de solo este repo
```

> [!warning] `user.email` es la identidad de los commits, no una cuenta
> Git no valida el correo: lo guarda en cada commit para siempre (autoría e integridad). Si lo escribís mal o usas el correo de otra persona, **cada commit queda atribuido equivocadamente** y corregirlo histórico excribe historia (ver [[Rebase]]). GitHub además verifica el correo contra los tuyos para vincular los commits a tu perfil: usa el correo noreply de GitHub si no querés exponer el real (`ID+usuario@users.noreply.github.com`). Corregir los últimos commits mal atribuidos: cambiar `user.email` y `git commit --amend --reset-author` (solo si no se compartieron).

## Configuración mínima inicial

```bash
git config --global user.name  "Tu Nombre"
git config --global user.email "tu@correo.com"
git config --global init.defaultBranch main    # ramas nuevas nacen como main, no master
```

`init.defaultBranch` evita el renombrado post-hoc `master → main` en cada repo nuevo (git ≤ 2.27 lo creaba como `master` y solo avisaba; desde 2.28 se respeta la config).

## Catálogo de claves esenciales

### Identidad y ramas

| Clave | Valores | Defecto | Notas |
|---|---|---|---|
| `user.name` | texto | — | Nombre en commits |
| `user.email` | texto | — | Correo en commits (ver aviso arriba) |
| `user.signingkey` | clave GPG/SSH | — | Clave usada al firmar (ver Firmas abajo) |
| `init.defaultBranch` | nombre de rama | `master` | Rama inicial de `git init` |

### Fin de línea: `core.autocrlf` y `core.eol`

El eterno dolor de los equipos mixtos Windows/Linux/Mac. Git guarda en el repositorio **LF** por convención y convierte según config:

| Sistema | `core.autocrlf` recomendado | Efecto al `add` | Efecto al `checkout` |
|---|---|---|---|
| Windows (con editores que usan CRLF) | `true` | CRLF → LF al indexar | LF → CRLF al restaurar |
| Linux / macOS | `input` | deja LF como está | no convierte (nunca escribe CRLF) |
| Proyectos estrictos multiplataforma | `false` + `.gitattributes` con `* text=auto` | nada | nada — la atribución manda (ver [[Archivos Especiales]]) |

Claves relacionadas: `core.eol` (`lf`/`crlf`/`native`, define el eol del working tree), `core.safecrlf` (`warn`/`true`/`false`, qué tan estricto es al convertir). **Recomendación 2026**: no confíes en `autocrlf` solo — declará `text=auto` en `.gitattributes` del proyecto, que gana sobre la config personal.

### Flujo de ramas e integración

| Clave | Valores | Defecto | Notas |
|---|---|---|---|
| `pull.rebase` | `false` / `true` / `interactive` / `merges` | `false` | ¿`git pull` mergea o rebasea? (ver [[Rebase]]) |
| `pull.rebase.autostash` | `true`/`false` | `false` | Stashea cambios sucios automáticamente antes del pull --rebase |
| `pull.ff` | `only`/`no`/`true` | `true` | `only` = nunca crea commit de merge en pull (fuerza fast-forward) |
| `push.default` | `nothing`/`current`/`upstream`/`simple`/`matching` | `simple` | Qué empuja `git push` sin args (ver tabla abajo) |
| `fetch.prune` | `true`/`false` | `false` | `true` = `git fetch -p` por defecto: borra refs de ramas remotas eliminadas |
| `rebase.autoStash` | `true`/`false` | `false` | Autostash en rebase interactivo |
| `rebase.instructionFormat` | template `%h %s` | — | Muestra el mensaje real en el editor de `rebase -i` |
| `merge.conflictstyle` | `merge`/`zdiff3` | `merge` | Formato de los marcadores de conflicto |

`push.default` en detalle:

| Valor | Comportamiento de `git push` sin argumentos |
|---|---|
| `simple` (defecto) | Pushea la rama actual a su upstream **solo si** el nombre coincide; si no, aborta con explicación |
| `upstream` | Pushea la rama actual a su upstream aunque los nombres difieran |
| `current` | Pushea la rama actual a una rama **del mismo nombre** en el remoto |
| `matching` | Pushea todas las ramas locales que tengan homónima en el remoto (el viejo defecto — peligroso, pushea de más) |
| `nothing` | Rechaza `git push` sin argumentos explícitos (paranoicos y flujos con scripts) |

`merge.conflictstyle zdiff3` (verificado) añade la sección base a los conflictos — mucho más claro que el estilo `merge` clásico:

```
l1
<<<<<<< HEAD
MAIN
||||||| 2eb4ef7        <-- versión base (común ancestro)
l2
=======
FEAT
>>>>>>> feat
```

### Herramientas externas

| Clave | Ejemplo | Notas |
|---|---|---|
| `core.editor` | `vim`, `code --wait`, `nano` | Editor para commits, rebase -i, `git commit` sin -m |
| `merge.tool` | `vimdiff`, `meld`, `code --wait`, `kdiff3` | Usado por `git mergetool` |
| `diff.tool` | `code --wait`, `vimdiff` | Usado por `git difftool` |
| `merge.tool` + `mergetool.keepBackup` | `false` | Evita los `.orig` sueltos tras mergetool |
| `color.ui` | `auto`/`always`/`false` | `auto` por defecto desde git 2.x |

### Atajos (alias)

```bash
git config --global alias.st status -sb
git config --global alias.lg "log --graph --oneline --decorate --all"
git config --global alias.co checkout
git config --global alias.unstage "restore --staged"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.amend "commit --amend --no-edit"
```

Los alias viven en `[alias]` del gitconfig; pueden invocarse como `git st`. Un alias puede expandir a otro comando (`alias.lg` arriba) o a un shell externo con `!` (ver [[Bash Scripting - Fundamentos Básicos]]).

### Rendimiento e integridad

| Clave | Valores | Defecto | Notas |
|---|---|---|---|
| `core.fsmonitor` | `true`/`false` | `false` | Watcher de archivos: acelera `status` en repos grandes |
| `core.untrackedCache` | `true`/`false` | `false` | Cachea la búsqueda de archivos sin trackear |
| `gc.auto` | número | 256788 | Cuándo corre el garbage collector solo |
| `rerere.enabled` | `true`/`false` | `false` | Recuerda resoluciones de conflictos (ver [[Ramas]]) |

## Autenticación: HTTPS vs SSH

| | HTTPS | SSH |
|---|---|---|
| Puerto | 443 (fuegogos amigables) | 22 (o 443 vía ssh.github.com) |
| Credenciales | Credential helpers (token/personal access token) | Llave privada + `ssh-agent` |
| Tokens rotan | Sí (PAT expira, fine-grained tokens) | Llave de larga duración |
| Múltiples cuentas | Configurable por host | Natural vía `~/.ssh/config` |
| Setup inicial | Mínimo | Generar e instalar llave |
| Corporativo/CI | Frecuentemente preferido | Frecuentemente preferido |

```bash
git remote add origin https://github.com/usuario/repo.git   # HTTPS
git remote add origin git@github.com:usuario/repo.git       # SSH
```

> [!warning] Los tokens son credenciales, no adornos
> Desde 2021 GitHub **no acepta contraseñas** para operaciones Git por HTTPS: solo tokens PAT/OAuth. Un PAT con alcance `repo` equivale a tu cuenta completa — guardalo en el credential helper, nunca en un repo o script. Los helpers: `store` guarda en claro en `~/.git-credentials` (solo máquinas personales de confianza), `cache` lo memoriza N segundos (por defecto 15 min), `manager` (Git Credential Manager, el estándar en Windows/macOS) los persiste en el llavero del sistema.

```bash
# Probar credenciales HTTPS
git ls-remote https://github.com/usuario/repo.git   # pide token o usa el helper
```

## Llaves SSH (ed25519)

```bash
ssh-keygen -t ed25519 -C "tu@correo.com"        # genera ~/.ssh/id_ed25519{,.pub}
eval "$(ssh-agent -s)"                           # levanta el agente
ssh-add ~/.ssh/id_ed25519                        # carga la llave (macOS: ssh-add --apple-use-keychain)
cat ~/.ssh/id_ed25519.pub                        # copiar al panel de GitHub → SSH keys
ssh -T git@github.com                           # verificar: "Hi usuario! You've successfully authenticated..."
```

ed25519 reemplazó a RSA por ser más corta y segura a la vez; si tu servidor viejo solo acepta RSA: `ssh-keygen -t rsa -b 4096 -C ...`.

### Varias cuentas / varios hosts: `~/.ssh/config`

```ssh-config
# Cuenta personal
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

# Cuenta laboral
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
```

Y en el repo laboral el remote usa el alias:

```bash
git remote set-url origin git@github-work:empresa/repo.git
git remote set-url --push origin git@github-work:empresa/repo.git
```

`Host github-work` es solo un alias local: para Git es un host distinto, para GitHub sigue siendo `github.com`. Patrón clásico de la duda "mis commits salen con la cuenta equivocada": dos remotos `origin` con hosts distintos en el mismo repo.

## Firmar commits y tags (GPG o SSH)

La firma criptográfica prueba que el commit salió de tu llave. GitHub muestra el badge "Verified":

| Esquema | Config | Verificación |
|---|---|---|
| GPG clásico | `gpg.format = openpgp` (defecto), `user.signingkey = HUELLA` | `git log --show-signature`, `git verify-commit <sha>` |
| SSH (moderno, más simple) | `gpg.format = ssh`, `user.signingkey = ~/.ssh/id_ed25519.pub`, `gpg.ssh.allowedSignersFile = ~/.config/git/allowed_signers` | igual |
| X.509 | `gpg.format = x509` | Corporativo raro |

```bash
git config --global user.signingkey ABCD1234
git config --global commit.gpgsign true      # firmar TODOS los commits (divisivo: molesta en CI/scripts)
git config --global tag.gpgsign true         # firmar tags (muy recomendado en releases, ver [[Tags]])
git commit -S -m "firmado"                    # firmar solo este commit
git verify-commit HEAD
```

`commit.gpgsign true` global choca con CI y commits automatizados: alternativa práctica — firmar solo tags (`tag.gpgsign true`) y los commits que exportas, vía `-S` puntual.

## `.gitignore` personal vs del proyecto: tres mecanismos

| Mecanismo | Archivo | ¿Se commitea? | Alcance | Uso típico |
|---|---|---|---|---|
| `.gitignore` | raíz del repo | **Sí** | Todo el que clone | Basura del proyecto: `node_modules`, `dist` |
| `.git/info/exclude` | `.git/info/exclude` | **No** (nunca sale del repo) | Solo tu clon | Cosas personales de ese repo: notas locales, archivos de tu IDE |
| `core.excludesFile` | `~/.config/git/ignore` (por defecto) | No | Todos tus repos | Basura personal universal: `.DS_Store`, `.idea/`, `*.swp` |

Verificado en la práctica: `echo "secreto.txt" > .git/info/exclude` hace que `secreto.txt` deje de aparecer en `git status` sin afectar a nadie más; `core.excludesFile` apunta a un archivo global con el mismo efecto. Las tres reglas se **apilan** (las tres fuentes suman). La sintaxis es idéntica — ver detalle completo en [[Archivos Especiales]].

## Hooks y linters: señalar el capítulo completo

Git ejecuta scripts que vos colocás en `.git/hooks/` (no viajan con el repo). El flujo moderno de equipo los empaqueta como dependencias:

| Herramienta | Qué aporta |
|---|---|
| **husky** (Node, ver [[Node JS]]) | Gestiona los hooks en equipos: se instala con el proyecto |
| **pre-commit** (Python) | Framework de hooks multi-lenguaje |
| **lint-staged** | Corre linters/formatters solo sobre los archivos staged |

Catálogo completo de hooks y patrones de uso en [[Archivos Especiales]].

## Recetas de arranque rápido

Setup limpio en máquina nueva:

```bash
git config --global user.name  "Tu Nombre"
git config --global user.email "tu@correo.com"
git config --global init.defaultBranch main
git config --global pull.rebase false          # o true si tu equipo vive en rebase
git config --global fetch.prune true
git config --global core.editor "code --wait"  # o vim/nano
git config --global alias.lg "log --graph --oneline --decorate --all"
# SSH:
ssh-keygen -t ed25519 -C "tu@correo.com" && ssh-add ~/.ssh/id_ed25519
# HTTPS: instalar Git Credential Manager o configurar helper
```

Auditar una máquina existente:

```bash
git config --list --show-origin | less     # ¿de dónde sale cada valor?
git config --global -e                     # editar global
git config --unset local rama vieja         # limpiar un valor local que pisa el global
```

## Errores comunes

1. **Configurar `user.email` con `--local`** en un repo personal que después publiques: el correo queda en la config commiteable... no, la config local vive en `.git/config` (no se commitea) — pero **copia de `.git/config` entre máquinas** sí arrastra identidades ajenas. Identidad siempre en `--global`.
2. **`core.autocrlf` distinto entre compañeros sin `.gitattributes`**: cada checkout convierte distinto y aparecen "cambios" de todo el archivo. Solución definitiva: `* text=auto` en `.gitattributes` (ver [[Archivos Especiales]]) + `git add --renormalize .`.
3. **Cambiar `user.email` y esperar que se corrija lo viejo** — los commits ya emitidos conservan el correo. Corrección solo de los últimos: `--amend --reset-author`; del histórico completo: reescribir (ver [[Rebase]]).
4. **`git push` rechazando con "refusing to update checked out branch"** — `push.default` no permite eso; o estás en detached HEAD. Leer el mensaje completo, que Git ya explica la causa y el fix.
5. **Alias que ocultan flags destructivos** (`alias.co "checkout -f"`): si un alias hace algo sorprendente, `git config --get alias.co` lo revela.
6. **Llave SSH que "no se usa"** — el agente no la tiene cargada (`ssh-add -l` para ver) o `~/.ssh/config` apunta a otro IdentityFile. Diagnóstico: `ssh -vT git@github.com`.

## Cheatsheet

| Quiero... | Comando |
|---|---|
| Ver toda la config + origen | `git config --list --show-origin` |
| Editar global/local | `git config --global -e` / `--local -e` |
| Identidad | `git config --global user.name / user.email` |
| Rama por defecto `main` | `git config --global init.defaultBranch main` |
| Pull con rebase | `git config --global pull.rebase true` |
| Borrar ramas remotas muertas al fetch | `git config --global fetch.prune true` |
| Conflictos con base | `git config --global merge.conflictstyle zdiff3` |
| Autostash en rebase | `git config --global rebase.autoStash true` |
| Recordar resoluciones de conflicto | `git config --global rerere.enabled true` |
| Ignorar archivos solo para mí | `.git/info/exclude` o `core.excludesFile` |
| Probar SSH | `ssh -T git@github.com` |
| Firmar un commit | `git commit -S -m "..."` |

## Relacionadas

- [[Qué es Git]] — el modelo de objetos y áreas que esta config gobierna
- [[Comandos Git]] — la referencia de comandos que usa todo esto
- [[Ramas]] — tracking branches, `pull.rebase` y flujos donde la config importa
- [[Rebase]] — por qué `pull.rebase` cambia tu día a día y cuándo activarlo
- [[Archivos Especiales]] — `.gitignore` vs exclude vs `core.excludesFile`, `.gitattributes` para eol, hooks y husky
- [[Bash Scripting - Fundamentos Básicos]] — aliases con `!` que invocan shell y scripts de setup
- [[Node JS]] — husky/lint-staged en el ecosistema Node
