---
tags:
  - ciberseguridad
  - herramientas
  - tls
tipo: herramienta
estado: terminado
---

# SSLyze — Analizador SSL/TLS

**SSLyze** es un analizador de configuración SSL/TLS diseñado por y para **automatización**: escanea rápido un servidor, lo descompone en *plugins* de escaneo (certificados, cifrados por versión, vulnerabilidades conocidas, resumption, ROBOT…) y produce un **JSON estructurado** pensado para pipelines. Es la herramienta que se usa cuando la auditoría TLS tiene que repetirse: en CI, en auditorías programadas o sobre miles de hosts.

> [!info] SSLyze vs SSLScan, en una frase
> [[SSLScan]] es el *estetoscopio*: rápido, legible, para mirar un servidor ahora. SSLyze es el *laboratorio*: mismo tipo de datos, pero con plugins nombrados, API de Python y JSON estable que se puede **assertar en CI**. Un pentest manual suele empezar con sslscan y termina con el JSON de sslyze adjunto al informe.

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado (`sslyze --version`) |
| pip (recomendado) | `pip3 install "sslyze"` |
| Actualizar firmas/sondas | `sslyze --update` (registra el server profile para scan commands) |
| Como librería | `pip3 install sslyze` → `from sslyze import Scanner` |

Requiere Python 3. No necesita root (no usa paquetes crudos; usa su propio stack TLS).

## Modelo de plugins (scan commands)

SSLyze 5.x organiza los escaneos como **scan commands** (una bandera = un plugin); sin banderas de escaneo, ejecuta el conjunto por defecto (*regular*). En 4.x existía `--regular` explícito — aún aparece en guías y notas viejas; en 5.x se sustituye por seleccionar plugins o dejar el default.

### Plugin de certificados (`--certificate-info`)

| Flag | Qué escanea |
|---|---|
| `--certificate-info` | Cadena completa, validez, SAN, claves, trust, OCSP stapling,CAA — el equivalente `--certinfo` de versiones previas |
| `--certinfo_full` (4.x) | Versión extendida; en 5.x: `--certificate-info` ya incluye la cadena completa |

### Plugins de vulnerabilidades (los "de negocio")

| Flag | Vulnerabilidad | CVE / nombre |
|---|---|---|
| `--heartbleed` | Memory disclosure en OpenSSL | CVE-2014-0160 |
| `--openssl_ccs` | Inyección CCS → manipulación del handshake | CVE-2014-0224 |
| `--robot` | Bleichenbacher/oráculo RSA PKCS#1 v1.5 en handshake | ROBOT (2017) — hermano conceptual de [[PadBuster - Herramienta de Ataque de Relleno Oracle]] |
| `--compression` | Compresión TLS (CRIME) | CVE-2012-4929 |
| `--reneg` | Renegociación insegura (client-initiated) | CVE-2009-3555 |
| `--fallback` | TLS Fallback SCSV (¿el servidor rechaza el downgrade forzado?) | CVE-2014-3566-related |
| `--resum` | Capacidad de *session resumption* (rendimiento/0-RTT) | — |
| `--early_data` | 0-RTT en TLS 1.3 (riesgo de replay) | — |

### Plugins de cifrados por versión

| Flag | Protocolo |
|---|---|
| `--sslv2` / `--sslv3` | SSL 2.0 / 3.0 (solo para confirmar que están **caídos**) |
| `--tlsv1` / `--tlsv1_1` | TLS 1.0 / 1.1 (deprecados, RFC 8996) |
| `--tlsv1_2` / `--tlsv1_3` | TLS 1.2 / 1.3 |
| `--cipher_list=<lista>` | Catálogo propio (`ALL`, `DEFAULT`, cadena OpenSSL) |
| `--mozilla_config=<old|intermediate>` | Escaneo + **compliance Mozilla** (5.3+): `old` o `intermediate` profiles |

> [!tip] `--mozilla_config` es la joya del CI
> Añade comprobación de cumplimiento de los perfiles Mozilla (old/intermediate) sobre el resultado y **devuelve código de salida ≠ 0** si no se cumple — es la forma nativa de romper el pipeline cuando alguien reintegra TLS 1.0 en el nginx.

### Plugins de conexión / comportamiento

| Flag | Qué hace |
|---|---|
| `--http_get` | GET tras el handshake: ¿redirige? ¿HTTPS-only? ¿headers HSTS? |
| `--starttls=<proto>` | `smtp`, `imap`, `pop`, `ftp`, `xmpp`, `xmpp_server`, `ldap`, `mssql`, `mysql`, `postgres`, `rmi`, `rdp`, `socks`, `nntp`, `none` |
| `--sni=<hostname>` | SNI explícito (obligatorio tras IP con virtual hosts) |
| `--https_tunnel=<host:puerto>` | Túnel HTTPS vía proxy CONNECT (escanear detrás de [[WFuzz]]/Burp) |
| `--nb_retries=<n>` | Reintentos por probe |
| `--timeout=<seg>` | Timeout de conexión (defecto ~5-10 s) |

### Salida y objetivos

| Flag | Significado |
|---|---|
| `--json_out=<archivo>` | JSON completo del escaneo (el artefacto canónico) |
| `--xml_out=<archivo>` | XML (legado) |
| `--targets_in=<archivo>` | Objetivos, uno por línea (`host`, `host:puerto`, `starttls://host:25`) |
| `--quiet` | Solo errores — ideal cuando lo único que importa es el JSON |
| `--update` | Actualiza el registro interno de scan commands |

## Escaneo regular vs completo

| Modo | Comando | Contenido |
|---|---|---|
| Regular (default) | `sslyze ejemplo.com` | Cifrados por versión, certificado, vulnerabilidades básicas — el 90 % de hallazgos |
| Dirigido | `sslyze --certificate-info --tlsv1_3 ejemplo.com` | Solo los plugins pedidos — rápido y fino |
| Completo | `sslyze --certificate-info --heartbleed --openssl_ccs --robot --compression --reneg --resum --fallback --http_get --tlsv1_3 ejemplo.com` | Todo el catálogo en una pasada |
| Batería | `sslyze --targets_in=hosts.txt --json_out=res.json --quiet` | Auditoría de n hosts para el informe |

## Salida de texto: interpretarla

```
  * DIGITAL CERTIFICADO
      CN=ejemplo.com  SAN=ejemplo.com,www.ejemplo.com
      Issuer: CN=R11,O=Let's Encrypt
      Validity: [2026-05-01, 2026-07-30]     ← caducidad: <30 días = alerta operativa
      Signature Algorithm: sha256WithRSAEncryption
  * ROBOT: VULNERABLE (padding oracle, respuesta "bleichenbacher")
  * TLS 1.3: 5 cipher suites, e.g. TLS_AES_256_GCM_SHA384 (OK)
  * TLS 1.2: 7 cipher suites, e.g. ECDHE-RSA-AES128-GCM-SHA256 (OK)
  * TLS 1.0: VULNERABLE (BEAST)  ← debe ser "not supported"
```

| Señal | Acción en informe |
|---|---|
| Versión aceptada con `VULNERABLE` | Hallazgo con CVE y prueba de concepto del escaneo |
| Cadenas incompletas / issuer ausente | El servidor no envía intermediaria → errores en clientes estrictos |
| `resum` no soportado | Degradación de rendimiento (handshake completo por conexión) |
| 0-RTT habilitado | Riesgo de replay en peticiones idempotentes — decisión de diseño |

## JSON: estructura y parsing con `jq`

Esquema (5.x; varía ligeramente entre versiones — inspeccionar con `jq 'keys'`):

```json
{
  "sslyze_options":  { "scan_commands": [...], "invalid_server_strings": [] },
  "sslyze_runtime":  { "total_scan_duration": 2.31 },
  "targets_results": [
    {
      "server_location": { "hostname": "ejemplo.com", "ip_address": "93.184.216.34", "port": 443 },
      "scan_status": "COMPLETED",
      "certificate_info_result": { "certificate_chain": [ { "subject": {...}, "issuer": {...}, "validity": {...} } ] },
      "tls_1_3_scan_result": { "cipher_suites": [ { "name": "TLS_AES_256_GCM_SHA384", ... } ] },
      "robot_result": { "robot_result": "VULNERABLE" },
      "openssl_ccs_result": { "openssl_ccs_injection": "NOT_VULNERABLE" },
      "http_headers_result": { "strict_transport_security_policy": {...} }
    }
  ]
}
```

Recetas de extracción:

```bash
# Cadenas de cifrados TLS 1.3 aceptados del primer target
jq -r '.targets_results[0].tls_1_3_scan_result.cipher_suites[].name' res.json

# ¿Algún target con ROBOT vulnerable?
jq -r '.targets_results[] | select(.robot_result.robot_result=="VULNERABLE")
       | .server_location.hostname' res.json

# Fechas de caducidad del certificado
jq -r '.targets_results[] | .certificate_info_result.certificate_chain[0]
       | "\(.subject.common_name) \(.validity.not_valid_on) → \(.validity.not_valid_after)"' res.json

# Resumen de fallos de escaneo (hosts que no completaron)
jq -r '.targets_results[] | select(.scan_status!="COMPLETED")
       | "\(.server_location.hostname): \(.scan_status)"' res.json

# Versiones soportadas por target (comprobación de deprecación)
for v in tls_1_0_scan_result tls_1_1_scan_result ssl_3_0_scan_result; do
  jq -r --arg v "$v" '.targets_results[] | "\(.server_location.hostname) \($v): \(.[$v] // "not supported")"' res.json
done
```

> [!warning] Claves con nombres que cambian entre 4.x y 5.x
> 4.x: `tlsv1_3_scan_result` con `cipher_suites`; 5.x: `tls_1_3_scan_result` (con guiones bajos dobles) y añade `scan_commands_performed`. Antes de basar un pipeline en el JSON, fijar la versión de sslyze en el `requirements` y validar el esquema con `jq 'paths' | head` sobre una salida real.

## Uso como librería de Python

```python
from sslyze import Scanner, ServerNetworkConfiguration, ServerConnectivityTester
from sslyze.plugins.scan_commands import ScanCommand

scanner = Scanner()
scanner.queue_scans(
    "ejemplo.com:443",
    [ScanCommand.CERTIFICATE_INFO, ScanCommand.HEARTBLEED, ScanCommand.TLS_1_3_SCAN],
)
for result in scanner.get_results():
    print(result.scan_status)
    print(result.certificate_info_result.certificate_chain[0].subject.common_name)
```

La API permite meter la auditoría TLS dentro de herramientas propias (p. ej. un auditor de infraestructura que combine Nmap + sslyze + informe LaTeX).

## Flujo de trabajo en CI

```yaml
# Ejemplo de pipeline: auditar los TLS públicos desplegados
- name: TLS audit
  run: |
    pip install sslyze
    sslyze --targets_in=hosts.txt --json_out=res.json --quiet --mozilla_config=intermediate
    # --mozilla_config!=compliant => exit != 0 => rompe el pipeline
```

| Práctica | Detalle |
|---|---|
| Fijar versión | El esquema JSON es estable **dentro** de una minor; fijar `sslyze==5.x.y` |
| Umbral de caducidad | `jq` sobre `not_valid_after` → fallar si < 21 días |
| Solo hallazgos nuevos | Guardar `res.json` por ejecución y diff de claves (`robot_result`, versiones aceptadas) |
| Artefacto de auditoría | Adjuntar el JSON al informe de cumplimiento ([[Creación de Informes en LaTeX]]) |

### Qué archivar en cada auditoría TLS

| Artefacto | Origen | Uso posterior |
|---|---|---|
| `res.json` | `--json_out` | Evidencia máquina; diff entre auditorías |
| Captura de texto | salida estándar con plugins | Figura del informe |
| Versión de sslyze + fecha | `sslyze --version` | Reproducibilidad de los hallazgos |
| Lista de objetivos | `targets_in` o salida de Nmap | Prueba de cobertura del alcance |

## Errores comunes (con mensajes exactos)

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `ServerConnectionError` / unable to connect | Timeout, puerto cerrado o firewall | Subir `--timeout`, verificar puerto con Nmap, comprobar SNI |
| Escaneo vacío tras proxy corporativo | Falta el túnel | `--https_tunnel=proxy:8080` |
| `--regular no es una opción válida` (5.x) | Flag de 4.x | Quitar la bandera (default) o seleccionar plugins |
| STARTTLS falla con "handshake error" | Servicio que requiere auth previo o protocolo equivocado | Probar otro `--starttls`; verificar con `openssl s_client -starttls` ([[OpenSSL Comandos Principales]]) |
| Sondas de vulnerabilidad dan NOT_VULNERABLE dudoso tras CDN | Escaneaste el edge, no el origen | `--sni` + hostname directo; considerar origen (bypass DNS) |
| `--update` falla sin red | Actualización del registro de scan commands requiere internet | Ejecutar en etapa de build con red o instalar imagen preactualizada |
| JSON parsea a `null` en las claves de vulnerabilidades | Target no completó (`scan_status` ≠ COMPLETED) | Filtrar primero por `.scan_status=="COMPLETED"` |

## Cheatsheet

```bash
sslyze ejemplo.com                        # regular (default)
sslyze --certificate-info ejemplo.com     # solo plugin de certificados
sslyze --certificate-info --heartbleed --openssl_ccs --robot --reneg --fallback ejemplo.com
sslyze --tlsv1_3 --cipher_list=ALL ejemplo.com
sslyze --starttls=smtp --certificate-info correo.local:25
sslyze --sni=app.local --https_tunnel=127.0.0.1:8080 app.local
sslyze --targets_in=hosts.txt --json_out=res.json --quiet
sslyze --mozilla_config=intermediate ejemplo.com      # compliance + exit != 0
jq -r '.targets_results[0].tls_1_3_scan_result.cipher_suites[].name' res.json
```

## Referencias

- Repositorio (nabla-c0d3): <https://github.com/nabla-c0d3/sslyze>
- Documentación de uso y JSON schema: <https://abalone12.github.io/sslyze/>
- Perfiles Mozilla: <https://wiki.mozilla.org/Security/Server_Side_TLS>
- Complemento rápido: [[SSLScan]]; diagnóstico manual: [[OpenSSL Comandos Principales]]

## Relacionadas

- [[SSLScan]] — vista rápida y legible del mismo terreno
- [[OpenSSL Comandos Principales]] — inspección de la cadena con `s_client`
- [[Nmap Guía Completa]] — `ssl-*` scripts para localizar puertos TLS
- [[PadBuster - Herramienta de Ataque de Relleno Oracle]] — el lado ofensivo del test ROBOT
- [[Creación de Informes en LaTeX]] — cómo empaquetar el JSON en un informe de cumplimiento
