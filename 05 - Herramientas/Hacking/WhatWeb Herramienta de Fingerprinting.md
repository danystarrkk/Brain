---
aliases: WhatWeb, Fingerprinting Web
tags:
  - hacking/herramientas
  - hacking/reconocimiento
estado: 🟢 Terminado
---
# WhatWeb: Herramienta de Fingerprinting Web

**WhatWeb** es una herramienta de fingerprinting de sitios web que identifica:

- Sistemas de Gestión de Contenido (CMS)
- Plataformas de blogging
- Frameworks JavaScript/CSS
- Servidores web
- Módulos de Apache
- Dispositivos de red
- Y más de 1800 plugins diferentes

> [!info] ¿Cómo funciona WhatWeb?
> WhatWeb utiliza una combinación de técnicas para identificar tecnologías web. Esto incluye el análisis de encabezados HTTP (como `Server`, `X-Powered-By`), el contenido HTML (meta tags, scripts, comentarios, rutas de archivos), patrones en URLs, y la detección de errores o respuestas específicas. Cada "plugin" de WhatWeb contiene un conjunto de firmas (signatures) que buscan estos patrones para identificar una tecnología específica.

## Tabla de Opciones Principales de WhatWeb

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-h` | Muestra ayuda | `whatweb -h` |
| `--help` | Ayuda avanzada | `whatweb --help` |
| `-v` | Modo verboso | `whatweb -v example.com` |
| `--version` | Muestra versión | `whatweb --version` |
| `-a` | Nivel de agresividad (1-4) | `whatweb -a 3 example.com` |
| `--aggression` | Nivel de agresividad | `whatweb --aggression 3` |
| `-i` | Archivo de entrada con URLs | `whatweb -i urls.txt` |
| `--input-file` | Archivo de entrada | `whatweb --input-file=urls.txt` |

### Opciones de Salida

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-q` | Modo silencioso | `whatweb -q example.com` |
| `--quiet` | Modo silencioso | `whatweb --quiet example.com` |
| `-v` | Salida verbosa | `whatweb -v example.com` |
| `--verbose` | Salida verbosa | `whatweb --verbose example.com` |
| `--color` | Salida con colores | `whatweb --color=always example.com` |
| `--no-errors` | Suprime mensajes de error | `whatweb --no-errors example.com` |
| `-o` | Salida a archivo | `whatweb -o results.txt example.com` |
| `--output` | Salida a archivo | `whatweb --output=results.txt` |

### Formatos de Salida

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--log-json` | Salida en JSON | `whatweb --log-json=results.json` |
| `--log-xml` | Salida en XML | `whatweb --log-xml=results.xml` |
| `--log-html` | Salida en HTML | `whatweb --log-html=results.html` |
| `--log-csv` | Salida en CSV | `whatweb --log-csv=results.csv` |
| `--log-brief` | Salida breve | `whatweb --log-brief=results.txt` |
| `--log-verbose` | Salida verbosa | `whatweb --log-verbose=results.txt` |

### Opciones de Escaneo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--url-prefix` | Prefijo para URLs | `whatweb --url-prefix=https://` |
| `--url-suffix` | Sufijo para URLs | `whatweb --url-suffix=/admin` |
| `--follow-redirect` | Sigue redirecciones | `whatweb --follow-redirect=always` |
| `--max-redirects` | Máximo de redirecciones | `whatweb --max-redirects=10` |
| `--user-agent` | User-Agent personalizado | `whatweb --user-agent="CustomUA"` |
| `--header` | Encabezados personalizados | `whatweb --header="Cookie: test=1"` |
| `--cookie` | Cookies | `whatweb --cookie="session=abc"` |
| `--proxy` | Usar proxy | `whatweb --proxy=127.0.0.1:8080` |
| `--proxy-user` | Credenciales de proxy | `whatweb --proxy-user=user:pass` |

> [!info] Uso de Proxies en WhatWeb
> La opción `--proxy` es fundamental para el anonimato y para escanear objetivos que pueden estar detrás de firewalls o sistemas de protección que bloquean IPs directas. Permite enrutar el tráfico a través de un servidor intermediario, como Burp Suite o un proxy SOCKS, lo que puede ser útil para interceptar y modificar solicitudes, o para evadir restricciones geográficas.

### Opciones de Plugins

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--list-plugins` | Lista todos los plugins | `whatweb --list-plugins` |
| `--info-plugins` | Información de todos los plugins | `whatweb --info-plugins` |
| `--search-plugins` | Buscar plugins | `whatweb --search-plugins=wordpress` |
| `--plugins` | Especificar plugins | `whatweb --plugins=wordpress,apache` |
| `--grep` | Filtrar salida por plugin | `whatweb --grep=wordpress` |
| `--custom-plugin` | Plugin personalizado | `whatweb --custom-plugin=plugin.rb` |

> [!tip] Explorando los Plugins de WhatWeb
> Utiliza `--list-plugins` para ver la vasta gama de tecnologías que WhatWeb puede identificar. Esto te dará una idea de la profundidad de su capacidad de fingerprinting. Si buscas información específica sobre cómo un plugin detecta una tecnología, `--info-plugins` puede ser muy útil para entender las firmas utilizadas.

### Opciones de Rendimiento

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--threads` | Número de hilos de ejecución | `whatweb --threads=10` |
| `--concurrency` | Nivel de concurrencia | `whatweb --concurrency=20` |
| `--timeout` | Tiempo de espera de conexión | `whatweb --timeout=30` |
| `--open-timeout` | Tiempo de espera de apertura | `whatweb --open-timeout=10` |
| `--read-timeout` | Tiempo de espera de lectura | `whatweb --read-timeout=20` |

## Niveles de Agresividad

| Nivel | Descripción | Comportamiento |
|---|---|---|
| `1` | Stealthy | Solo solicitudes iniciales, pasivo |
| `2` | Default | Algunas solicitudes adicionales |
| `3` | Aggressive | Solicitudes más intrusivas |
| `4` | Heavy | Máxima intrusividad, muchas solicitudes |

> [!warning] Consideraciones sobre la Agresividad
> Los niveles de agresividad más altos (3 y 4) implican un mayor número de solicitudes HTTP y pueden incluir peticiones a rutas comunes de archivos, intentos de detección de errores o incluso la ejecución de JavaScript en algunos casos. Esto puede generar ruido en los logs del servidor objetivo y, en entornos de producción, podría activar sistemas de detección de intrusiones (IDS/IPS) o incluso causar una denegación de servicio si el servidor es vulnerable o está sobrecargado. Siempre utiliza los niveles de agresividad con precaución y solo en entornos autorizados.
>
> [!tip] Equilibrio entre Agresividad y Detección
> Para un reconocimiento inicial, el nivel `2` (Default) suele ser suficiente para obtener una buena cantidad de información sin ser demasiado ruidoso. Si necesitas más detalles y estás en un entorno de laboratorio o con permiso explícito, puedes aumentar a `3` o `4`. Recuerda que un mayor nivel de agresividad no siempre garantiza una detección más precisa, pero sí aumenta la probabilidad de identificar tecnologías menos obvias.