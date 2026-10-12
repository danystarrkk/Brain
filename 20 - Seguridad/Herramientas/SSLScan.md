---
tags:
  - ciberseguridad
  - herramientas
  - sslscan
tipo: herramienta
estado: terminado
---

# SSLScan

**SSLScan** es un escáner SSL/TLS de línea de comandos: fuerza el handshake con cada versión y cada cipher del catálogo de su OpenSSL local y reporta **qué acepta realmente el servidor**, junto con el certificado y tiempos de respuesta. Su nicho es la **rapidez y legibilidad**: en segundos produce la lista completa de cifrados aceptados, marcando en rojo los débiles, que es exactamente lo que se pone en un informe.

> [!info] Qué responde SSLScan
> "¿Qué protocolos acepta este servidor? ¿Qué cifrados dentro de cada uno? ¿Esos cifrados son débiles? ¿Qué certificado presenta y es válido? ¿Cuánto tarda el handshake?". Nada más y nada menos: para vulnerabilidades específicas (Heartbleed, ROBOT, CCS), CI automatizado o JSON estructurado, la herramienta hermana es [[SSLyze Analizador SSL TLS]]; para diagnóstico puntual de una conexión, [[OpenSSL Comandos Principales]] con `s_client`.

## SSLScan vs SSLyze vs testssl.sh

| Aspecto | **SSLScan** | [[SSLyze Analizador SSL TLS]] | testssl.sh |
|---|---|---|---|
| Velocidad | Muy rápida (handshakes directos) | Rápida, con plugins | Lenta (Bash) |
| Cifrados por protocolo | Sí — tabla Accepted/Preferred | Sí (por versión) | Sí |
| Vulnerabilidades (Heartbleed, ROBOT…) | Parcial (Heartbleed/CCS básicos) | **Sí, dedicada** (`--heartbleed`, `--robot`, `--openssl_ccs`) | Sí, más |
| Certificado | Completo (`--show-certificate`) | Completo (`--certinfo_full`) | Completo |
| Salida máquina | XML, JSON | **JSON rico y estable** (ideal CI) | JSON/CSV |
| STARTTLS | smtp, pop, imap, xmpp, ldap, ftp, rdp | Más protocolos + túnel proxy | Amplio |
| Veredicto A–F | No lo emite (hay que interpretarlo) | No lo emite | **Sí (gradeletters)** |
| Mejor para | Diagnóstico rápido, informes legibles | Auditoría programática, CI, JSON | Batería completa |

## Protocolos: qué significa cada hallazgo

| Protocolo | Estado | Comentario en informe |
|---|---|---|
| SSLv2 / SSLv3 | **Crítico** (POODLE, CCS, cracking de clave) | Hallazgo de severidad alta — deshabilitar ya |
| TLS 1.0 / 1.1 | **Obsoleto** (BEAST, sin AEAD, deprecados por RFC 8996) | Hallazgo medio-alto |
| TLS 1.2 | Estándar vigente si cifrados fuertes | Aceptable |
| TLS 1.3 | Recomendado (1-RTT, AEAD obligatorio, 5 cifrados) | Objetivo |

### Cifrados débiles a buscar en la salida

| Familia | Por qué es débil |
|---|---|
| `NULL` / `eNULL` | Sin cifrado — el handshake "funciona" pero todo va en claro |
| `EXPORT` / `DES-CBC3` (3DES) | Claves exportables / 64 bits efectivos (SWEET32) |
| `RC4` | Sesgos estadísticos (Bar Mitzvah, Royal Holloway) |
| `CBC` en TLS 1.0-1.2 con SHA1 | Oráculos de padding (BEAST, Lucky 13) — preferir AEAD |
| Cualquier cipher de **< 128 bits** | Clave demasiado corta |
| DHE/ECDHE con grupos < 2048 bits o sin firma | Weak DH (Logjam) |

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado (`sslscan --version`) |
| Debian/Ubuntu | `sudo apt install sslscan` |
| Compilar | `git clone https://github.com/rbsec/sslscan && cd sslscan && make static && sudo make install` |
| Docker | `docker run -it rbsec/sslscan <host>` |

Compilado contra su propio OpenSSL estático — no depende del TLS del sistema.

## Referencia de flags

### Básicos y objetivo

| Flag | Significado | Ejemplo |
|---|---|---|
| `--help` / `--version` | Ayuda / versión | |
| `<host>[:puerto]` | Objetivo posicional (puerto defecto 443) | `sslscan ejemplo.com:8443` |
| `--port=<puerto>` | Puerto explícito | `--port=993` |
| `--targets=<archivo>` | Lista de objetivos, uno por línea (`host[:puerto]`) | `--targets=lista.txt` |
| `--timeout=<seg>` | Timeout de conexión (defecto 3) | `--timeout=8` |
| `--sleep=<ms>` | Pausa entre pruebas (bajar ruido/WAF) | `--sleep=500` |
| `--connect-timeout=<seg>` | Timeout específico del TCP connect | |
| `--ip=<ip>` | Conectar a IP forzada (DNS name + IP distinto) | `--ip=10.10.10.50` |
| `--sni=<nombre>` / `--no-sni` | Forzar u omitir SNI | `--sni=app.ejemplo.com` |
| `--use-sni` | SNI con el hostname dado | |

### STARTTLS y proxy

| Flag | Significado | Ejemplo |
|---|---|---|
| `--starttls=<proto>` | Negociar STARTTLS antes del handshake: `ftp`, `smtp`, `pop`, `imap`, `xmpp`, `ldap`, `xmpp-server`, `postgres`, `mysql`, `rdp` | `--starttls=smtp` |
| `--proxy=<host:puerto>` | Escanear a través de proxy HTTP | `--proxy=127.0.0.1:8080` |
| `--proxy-auth=<user:pass>` | Credenciales del proxy | |

### Selección de protocolos

| Flag | Fuerza esa versión (por defecto prueba todas las disponibles) |
|---|---|
| `--ssl2` / `--ssl3` | SSLv2 / SSLv3 |
| `--tls1` / `--tls1-1` | TLS 1.0 / 1.1 |
| `--tls1-2` / `--tls1-3` | TLS 1.2 / 1.3 |
| `--groups=<lista>` | Grupos de intercambio (curvas) a probar: `--groups=P-256,X25519` |
| `--curves=<lista>` | Alias histórico de `--groups` en versiones previas |

### Client certificates y salida

| Flag | Significado | Ejemplo |
|---|---|---|
| `--cert=<archivo>` / `--pk=<archivo>` | Certificado y clave privada de cliente (mTLS) | `--cert=cli.pem --pk=cli.key` |
| `--xml=<archivo>` | Salida XML | `--xml=resultado.xml` |
| `--json=<archivo>` | Salida JSON | `--json=resultado.json` |
| `--pretty` | JSON indentado | |
| `--no-failed` | Oculta los ciphers rechazados (solo aceptados) — limpia el informe | |
| `--no-colour` / `--colour` | Sin color (para pipes/informes) | |
| `--show-certificate` | Certificado completo (texto) | |
| `--show-times` | Tiempos de handshake por cipher (detección de estado/latencia) | |
| `--show-sigs` | Firmas/algoritmos soportados por el certificado | |
| `--verbose` / `-v` | Detalle de depuración | |
| `--openssl` | Info del OpenSSL usado por sslscan | |
| `--hint` | Adivina el servidor real detrás de CDNs/proxies (p. ej. detectar backend) | |

## Interpretación de la salida

```
Accepted  TLSv1.2  256 bits  ECDHE-RSA-AES256-GCM-SHA384
Preferred TLSv1.3  256 bits  TLS_AES_256_GCM_SHA384         PFS: 256 bits, x25519, 256 bit MD5
Rejected  TLSv1.0  128 bits  DES-CBC3-SHA
```

| Marcador / columna | Significado |
|---|---|
| `Accepted` | El servidor **completó** el handshake con ese cipher → es lo que un cliente podrá usar |
| `Preferred` | Cipher que el servidor elige **primero** si el cliente lo soporta (por protocolo) |
| `Rejected` | Handshake fallido — el cipher está en el catálogo local pero el server lo descarta |
| bits | Fortaleza de la clave simétrica — rojo si < 128 (o < 256 en auditorías estrictas) |
| `PFS: ...` | Perfect Forward Secrecy: grupo DHE/ECDHE y tamaño — sin PFS = hallazgo |
| Bloque `Server Certificate` | CN/SAN, issuer, fechas, firma — mirar caducidad y CN ≠ SAN |
| `Qualys SSL Labs` line | En versiones recientes puede imprimir referencia de análisis complementario |

### Convirtiendo la salida en un veredicto (los grados no los emite la herramienta)

| Grado (convenio de informe) | Condición observada en sslscan |
|---|---|
| **A** | Solo TLS 1.2 AEAD + TLS 1.3, PFS con ≥ 2048/X25519, certificado válido |
| **B** | TLS 1.2 con algún cipher CBC "tolerable", sin legacy |
| **C** | TLS 1.0/1.1 activo o 3DES aceptado |
| **F** | SSLv3/SSLv2, RC4, EXPORT, NULL o claves < 128 bits aceptadas |

Para el grado oficial de la industria (A+ de Qualys) usar testssl.sh/SSL Labs; en informes de pentest el veredicto en prosa + captura de sslscan es lo estándar.

### Ejemplo de salida completa y su transcripción a hallazgos

```text
Accepted  TLSv1.2  256 bits  ECDHE-RSA-AES256-GCM-SHA384         PFS: 256 bits, secp384r1
Accepted  TLSv1.2  128 bits  ECDHE-RSA-AES128-GCM-SHA256         PFS: 256 bits, x25519
Accepted  TLSv1.2  112 bits  ECDHE-RSA-DES-CBC3-SHA              SSL3: DES-CBC3-SHA
Accepted  TLSv1.0  128 bits  ECDHE-RSA-AES128-SHA

Server Certificate:
    Subject:  CN=*.ejemplo.com
    Issuer:   C=US, O=DigiCert, CN=GeoTrust RSA CA 2048 G2
    Not valid before: 2026/01/10
    Not valid after:  2026/04/10
```

| Hallazgo de la salida | Hallazgo del informe |
|---|---|
| `Accepted TLSv1.0 ...` | **Medio**: protocolo deprecado (RFC 8996) aceptado — cifrar recomendación: deshabilitar |
| `DES-CBC3-SHA` (112 bits) | **Medio**: 3DES aceptado → SWEET32; eliminar del cipher string |
| `Not valid after: 2026/04/10` | **Operativo**: certificado con <30 días → alerta de renovación |
| `Accepted TLSv1.2 ... GCM` | Sin hallazgo — cifrados AEAD con PFS correctos |

Los `Accepted` de versiones legacy son el hallazgo: la existencia de **un solo cipher** aceptado en TLS 1.0 ya habilita el protocolo para cualquier cliente.

## Flujos de trabajo

### 1. Auditoría rápida de un host

```bash
sslscan --show-certificate --no-failed ejemplo.com:443
```

Solo lo que se acepta + el certificado: la vista de informe por defecto.

### 2. Batería de objetivos desde lista (de Nmap)

```bash
grep -E "443/tcp|8443/tcp" escaneo.gnmap | awk '{print $2":443"}' | sort -u > tls-hosts.txt
sslscan --targets=tls-hosts.txt --no-failed --json=tls.json --timeout=5
```

### 3. Correo con STARTTLS

```bash
sslscan --starttls=smtp --show-certificate correo.ejemplo.com:25
sslscan --starttls=imap  --show-certificate correo.ejemplo.com:993
```

### 4. Comprobar detrás de un proxy (pentest con Burp)

```bash
sslscan --proxy=127.0.0.1:8080 --no-colour ejemplo.com:443
```

### 5. Matar versiones legacy confirmadas

```bash
sslscan --tls1   ejemplo.com:443 | grep -m1 Accepted   # idealmente: (no output)
sslscan --ssl3   ejemplo.com:443 | grep -m1 Accepted
sslscan --tls1-1 ejemplo.com:443 | grep -m1 Accepted
```

Si aparece `Accepted`, la versión sigue viva → hallazgo.

## Errores comunes (con mensajes exactos)

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `Failed to connect to host` | Puerto/TLS incorrecto, host caído o firewall | Comprobar puerto (`[[Nmap Guía Completa]] -sV`); ¿STARTTLS?; subir `--timeout` |
| `Unknown STARTTLS protocol: X` | Nombre de protocolo no soportado | Usar uno de la lista (`smtp`, `imap`, `pop`, `ldap`, `xmpp`, `ftp`, `rdp`) |
| Todos los protocols `failed handshake` con servidor viejo | Servidor exige SNI o cipher fuera del build | `--sni=<hostname>`; recompilar sslscan actualizado |
| Resultados distintos a los del navegador | El navegador envía SNI y ALPN; sslscan por defecto no siempre | Añadir `--sni` y comparar con `openssl s_client -servername` |
| Salida con control de color en informes/instrumentación | Colores ANSI en la captura | `--no-colour` |
| El JSON no tiene la estructura esperada entre versiones | El esquema cambió entre 1.x y 2.x | Fijar versión; parsear con tolerancia o migrar a SSLyze para CI |

## Cheatsheet

```bash
sslscan ejemplo.com                                  # todo, 443
sslscan --show-certificate --no-failed ejemplo.com   # informe limpio
sslscan --port=8443 --ip=10.10.10.50 app.local
sslscan --starttls=smtp --show-certificate mail.local:25
sslscan --targets=hosts.txt --json=out.json --no-failed
sslscan --tls1-3 --no-failed ejemplo.com             # solo TLS1.3
sslscan --no-colour --no-failed ejemplo.com | grep Accepted
sslscan --sni=app.local --show-times 10.10.10.50
```

## Referencias

- Repositorio oficial: <https://github.com/rbsec/sslscan>
- Man/receta de Kali: <https://tools.kali.org/information-gathering/sslscan>
- Protocolos deprecados (RFC 8996): <https://datatracker.ietf.org/doc/html/rfc8996>
- Diagnóstico de una conexión concreta: [[OpenSSL Comandos Principales]]
- Auditoría profunda y JSON: [[SSLyze Analizador SSL TLS]]

## Relacionadas

- [[SSLyze Analizador SSL TLS]] — vulnerabilidades dedicadas y CI
- [[OpenSSL Comandos Principales]] — inspección manual con `s_client`
- [[Nmap Guía Completa]] — `--script ssl-enum-ciphers` para el primer sondeo
- [[Reconocimiento y Escaneo de Redes y Web]] — fase donde entra esta auditoría
