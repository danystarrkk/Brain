---
tags:
  - ciberseguridad
  - herramientas
  - openssl
tipo: herramienta
estado: terminado
---

# OpenSSL — Comandos Principales

**OpenSSL** es a la vez (1) la biblioteca criptográfica *de facto* de todo el software de red — Apache, Nginx, OpenSSH, Postfix, Python, curl — y (2) la CLI de referencia para **generar claves y certificados, inspeccionar TLS, verificar cadenas y cifrar archivos**. Para un pentester es la herramienta de diagnóstico TLS definitiva: cuando `curl -v` no dice qué pasó en el handshake, `openssl s_client` lo dice.

> [!info] Dos caras de una misma herramienta
> El binario `openssl` ejecuta *subcomandos* (`req`, `x509`, `s_client`, `enc`…) que son interfaces de la biblioteca subyacente. Cada versión añadió subcomandos nuevos (`genpkey`, `pkeyutl` modernizan a `genrsa`/`rsautl`). Regla: **usar los comandos modernos** (`genpkey`, `pkeyutl`) salvo cuando se reproduzca un apunte o guía antigua.

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux / Debian / Ubuntu** | Preinstalado (`openssl version`) |
| RHEL/Fedora | `sudo dnf install openssl` |
| Windows | Vendido con Git for Windows / LibreSSL; oficial: slproweb.com builds |

Sin dependencias externas: la CLI funciona en cualquier Linux mínimo.

## Subcomandos esenciales (mapa rápido)

| Subcomando | Función |
|---|---|
| `genpkey` / `genrsa` | Generar clave privada (moderna / legacy) |
| `req` | Crear CSR o certificado autofirmado |
| `x509` | Inspeccionar, convertir, firmar certificados |
| `verify` | Verificar cadena de confianza |
| `s_client` | Cliente TLS interactivo — diagnóstico y depuración |
| `s_server` | Servidor TLS de prueba |
| `s_time` | Medir handshake/rendimiento TLS |
| `ciphers` / `ciphersuites` | Listar suites disponibles (cliente) |
| `enc` | Cifrar/descifrar archivos (simétrico) |
| `dgst` | Hashes y firmas digitales |
| `pkeyutl` / `rsautl` | Operaciones con claves (cifrar, firmar) |
| `pkcs12` | Exportar/importar paquetes clave+cert (PFX/P12) |
| `pkcs8` | Convertir formatos de clave privada |
| `rand` | Generar bytes aleatorios |
| `ca` | CA rudimentaria (laboratorio) |

## Generación de claves

```bash
# RSA moderno (el comando recomendado hoy)
openssl genpkey -algorithm RSA -out clave.pem -pkeyopt rsa_keygen_bits:4096

# RSA legacy (equivalente clásico — aún muy visto en guías)
openssl genrsa -out clave.pem 4096

# EC (ECC) — más rápida y más pequeña que RSA a igual seguridad
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out clave_ec.pem

# Extraer la pública de una clave privada
openssl pkey -in clave.pem -pubout -out clave_pub.pem
```

| Tamaño/curva | Estado (2026) |
|---|---|
| RSA 2048 | Mínimo aceptable |
| RSA 4096 | Recomendado para CA y servicios largos |
| RSA < 2048 | Inseguro, rechazado por navegadores |
| EC P-256 / P-384 | Recomendado (rápido en handshake, clave corta) |
| DSA | Obsoleto — no usar |

## CSR y certificados autofirmados

```bash
# 1. CSR desde clave existente (interactivo: pide los campos)
openssl req -new -key clave.pem -out peticion.csr

# 2. CSR con subject en una sola línea (automatización)
openssl req -new -key clave.pem -out peticion.csr \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=MiEmpresa/OU=IT/CN=app.ejemplo.com"

# 3. CSR + clave en un solo paso (muy habitual en lab)
openssl req -new -newkey rsa:2048 -nodes -keyout clave.pem -out peticion.csr \
  -subj "/CN=app.ejemplo.com"

# 4. Certificado autofirmado válido 1 año (labs, servidores internos)
openssl req -x509 -newkey rsa:2048 -nodes -keyout clave.pem -out cert.pem \
  -days 365 -subj "/CN=app.ejemplo.com"

# 5. Autofirmado con SAN (Subject Alternative Name) — OBLIGATORIO hoy:
#    los navegadores rechazan CN sin SAN desde 2020
openssl req -x509 -newkey rsa:2048 -nodes -keyout clave.pem -out cert.pem -days 365 \
  -subj "/CN=app.ejemplo.com" \
  -addext "subjectAltName=DNS:app.ejemplo.com,DNS:localhost,IP:127.0.0.1"
```

| Campo del subject | Significado |
|---|---|
| `CN` | Common Name — el dominio (históricamente el hostname) |
| `O` / `OU` | Organización / departamento |
| `C` / `ST` / `L` | País / estado / localidad |

> [!warning] CN solo ya no basta
> Desde febrero de 2020 los navegadores **ignoran** el CN y exigen el campo `SAN` en el certificado. Un autofirmado con `-subj /CN=x` sin `-addext subjectAltName=...` da error `ERR_CERT_COMMON_NAME_INVALID` — es la trampa nº 1 de los certificados de laboratorio.

## Inspección de certificados

```bash
# Texto completo (todo lo que el certificado declara)
openssl x509 -in cert.pem -text -noout

# Resumen de una CSR (qué pide el solicitante)
openssl req -in peticion.csr -text -noout

# Fechas y subject compactos (para scripts)
openssl x509 -in cert.pem -noout -subject -dates -issuer

# Extraer la pública de un certificado
openssl x509 -in cert.pem -pubkey -noout

# Huella del certificado (lo que se compara al aprobar en navegador)
openssl x509 -in cert.pem -noout -fingerprint -sha256

# Serial y versión
openssl x509 -in cert.pem -noout -serial -dates
```

### Qué mirar en `x509 -text` (checklist)

| Sección del texto | Qué verificar |
|---|---|
| `Issuer` vs `Subject` | ¿Iguales? → autofirmado (o CA propia) |
| `Validity: Not Before / Not After` | Caducado o a punto de caducar |
| `Subject Alternative Name` | Que cubra todos los hostnames servidos |
| `Public Key Algorithm` + tamaño | RSA ≥ 2048, EC P-256+ |
| `Signature Algorithm` | `sha256WithRSAEncryption` o mejor; MD5/SHA1 = crítico |
| `X509v3 Basic Constraints` | `CA:TRUE` — quién puede firmar cadenas |
| `Key Usage` / `Extended Key Usage` | Que las usos permitidos coincidan con el rol (serverAuth, clientAuth) |
| `Authority Information Access` | Dónde descargar la cadena (OCSP/CRL) |

## Verificación de cadenas de confianza

```bash
# Verificar contra el bundle del sistema
openssl verify cert.pem

# Verificar con la CA emisora y su intermediaria (cadena completa)
openssl verify -CAfile ca.pem -untrusted cadena_intermedia.pem cert.pem

# Verificar un servidor en vivo (compara la cadena que presenta)
openssl s_client -connect ejemplo.com:443 -CAfile ca.pem </dev/null
```

Salida de `verify`: `cert.pem: OK` o errores exactos — `unable to get local issuer certificate` (falta la intermediaria en la cadena que envía el servidor), `certificate has expired`, `self-signed certificate`.

## `s_client` — el estetoscopio TLS

```bash
# Diagnóstico completo de un servidor HTTPS
openssl s_client -connect ejemplo.com:443 -showcerts

# Forzar versión (verificar que TLS 1.0/1.1/SSLv3 están caídos)
openssl s_client -connect ejemplo.com:443 -tls1_2
openssl s_client -connect ejemplo.com:443 -tls1_3
openssl s_client -connect ejemplo.com:443 -no_tls1_3

# Forzar un cipher concreto (probar si el servidor lo acepta)
openssl s_client -connect ejemplo.com:443 -cipher 'ECDHE-RSA-AES128-GCM-SHA256'

# SNI correcto en virtual hosts (obligatorio con IPs múltiples)
openssl s_client -connect 10.10.10.50:443 -servername app.ejemplo.com

# Servicio STARTTLS (correo, LDAP, etc.)
openssl s_client -starttls smtp -connect correo.ejemplo.com:25 -showcerts
openssl s_client -starttls imap  -connect correo.ejemplo.com:143
openssl s_client -starttls ldap  -connect dc.ejemplo.com:389
openssl s_client -starttls pop   -connect correo.ejemplo.com:110

# Verificar OCSP stapling
openssl s_client -connect ejemplo.com:443 -status </dev/null

# Enviar datos por la conexión (probar la aplicación de arriba)
echo -e "GET / HTTP/1.1\r\nHost: ejemplo.com\r\n\r\n" | openssl s_client -connect ejemplo.com:443 -quiet
```

### Interpretar la salida de `s_client`

| Bloque de salida | Qué dice |
|---|---|
| `Certificate chain` (con `-showcerts`) | Certificado del servidor + intermediarias — comprobar orden y completitud |
| `subject= / issuer=` | Identidad presentada vs emisora |
| `New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384` | Versión y cipher **negociados** de la conexión |
| `Verify return code: 0 (ok)` | Resultado de la validación con los CA configurados — ≠ 0 = cadena rota |
| `OCSP response:` | Stapling: si dice `No response` el servidor no lo entrega |

> [!tip] La trampa del EOF
> `s_client` espera la conexión abierta. En pipes y scripts **siempre** terminar con `</dev/null` o `echo |`, o el comando se queda colgado indefinidamente sin error.

## Cifrado y descifrado de archivos (simétrico)

```bash
# AES-256-CBC con clave derivada de contraseña (Salted)
openssl enc -aes-256-cbc -salt -in informe.txt -out informe.enc -pass pass:'MiClave'
openssl enc -aes-256-cbc -d -in informe.enc -out informe.txt -pass pass:'MiClave'

# Con clave de archivo en vez de passphrase (automatización, menos logs)
openssl enc -aes-256-cbc -salt -in datos.tar.gz -out datos.enc -pass file:clave.key

# AES-256-GCM (AEAD: autentica además de cifrar — disponible desde OpenSSL 3.x)
openssl enc -aes-256-gcm -salt -in f.txt -out f.enc -pass pass:clave

# Hash de un archivo (integridad)
openssl dgst -sha256 archivo.iso
openssl dgst -sha256 -hmac "secreto" archivo

# Generar material aleatorio (tokens, salts)
openssl rand -hex 32
```

| Algoritmo | Estado |
|---|---|
| `aes-256-cbc` | Clásico funcional; sin autenticación (usar con `-md sha256`) |
| `aes-256-gcm` | Recomendado (AEAD) en OpenSSL 3.x |
| `des`/`bf`/`rc4` | Obsoletos — `enc` los sigue listando, **no usarlos** |

> [!warning] `enc` sin `-md sha256` en OpenSSL 1.1.1+
> OpenSSL cambió el hash por defecto de MD5 a SHA-256 en los KDF. Un archivo cifrado con OpenSSL antiguo **no se descifra** con el nuevo sin añadir `-md md5` (y viceversa). Mensaje típico: `bad decrypt` / `error:06065064:digital envelope routines:EVP_DecryptFinal_ex:bad decrypt`. Guarda siempre junto al .enc la versión de OpenSSL usada.

## Firmas digitales y operaciones asimétricas

```bash
# Firmar con privada
openssl dgst -sha256 -sign clave_privada.pem -out firma.bin documento.txt

# Verificar con pública
openssl dgst -sha256 -verify clave_pub.pem -signature firma.bin documento.txt
# -> Verified OK

# Cifrar con pública / descifrar con privada (solo datos pequeños, ≤ tamaño de clave)
openssl pkeyutl -encrypt -pubin -inkey clave_pub.pem -in plano.txt -out cifrado.bin
openssl pkeyutl -decrypt -inkey clave_privada.pem -in cifrado.bin -out plano.txt
```

## Paquetes de certificados (conversión de formatos)

```bash
# PEM -> PKCS#12 (paquete para Windows/navegadores; pide passphrase)
openssl pkcs12 -export -out certificado.pfx -inkey clave.pem -in cert.pem -name "MiCert"

# PKCS#12 -> PEM
openssl pkcs12 -in certificado.pfx -out certificado.pem -nodes

# PEM (clave) -> PKCS#8 sin cifrar (formato moderno)
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in clave.pem -out clave8.pem

# DER binario -> PEM
openssl x509 -inform DER -in cert.der -outform PEM -out cert.pem

# Extraer solo la pública de un P12
openssl pkcs12 -in certificado.pfx -clcerts -nokeys -out cert_solo.pem
```

| Formato | Extensión típica | Contenido |
|---|---|---|
| PEM | `.pem`, `.crt`, `.key`, `.csr` | Base64 con encabezados `-----BEGIN ...-----` |
| DER | `.der`, `.crt` binario | Binario X.509/clave |
| PKCS#12 | `.p12`, `.pfx` | Binario: clave privada + cert + cadena, con contraseña |
| PKCS#8 | `.pk8` | Clave privada en estandarización moderna |

## Rendimiento y enumeración de ciphers

```bash
# Benchmarks: handshake y transferencia contra un servidor
openssl s_time -connect ejemplo.com:443 -new

# Benchmark local de algoritmos
openssl speed -evp aes-256-gcm rsa2048

# Suites disponibles en este build del cliente (TLS1.2 y anteriores)
openssl ciphers -v 'ALL:eNULL' | head
openssl ciphers -v -s -tls1_2

# TLS 1.3 usa otro nombre de catálogo
openssl ciphersuites

# Listar algoritmos soportados (OpenSSL 3.x)
openssl list -cipher-algorithms
openssl list -public-key-algorithms
```

> [!info] `ciphers` es la vista del cliente local
> `openssl ciphers` lista lo que **tu build** soporta, no lo del servidor. Para las suites del servidor usar `s_client` (cipher negociado) o directamente [[SSLScan]] / [[SSLyze Analizador SSL TLS]] (catálogo completo servidor).

## Recetas del día a día (pentest/lab)

```bash
# 1. Certificado autofirmado con SAN para un lab en 30 segundos
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 90 \
  -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# 2. Auditar qué TLS acepta un servidor (y si responde por IP con SNI)
openssl s_client -connect 10.10.10.50:443 -servername target.local </dev/null

# 3. Extraer el certificado de un servicio para análisis offline
openssl s_client -connect ejemplo.com:443 </dev/null 2>/dev/null | \
  openssl x509 -out cert.pem
openssl x509 -in cert.pem -noout -text | grep -A2 "Validity"

# 4. Comprobar si TLS 1.0/1.1 sigue activo (hallazgo de informe)
openssl s_client -connect ejemplo.com:443 -tls1  </dev/null   # debe fallar
openssl s_client -connect ejemplo.com:443 -tls1_1 </dev/null  # debe fallar

# 5. Cadena de correo con STARTTLS
openssl s_client -starttls smtp -connect correo.local:25 -showcerts </dev/null

# 6. P12 para importar en Windows desde un lab Linux
openssl pkcs12 -export -out lab.p12 -inkey key.pem -in cert.pem -name lab

# 7. Verificar una cadena entregada por un servidor interno
echo | openssl s_client -connect dc.local:636 -showcerts 2>/dev/null | \
  sed -n '/BEGIN CERT/,/END CERT/p' > servidor.pem
openssl verify -CAfile ca_interna.pem servidor.pem

# 8. Token aleatorio para una cookie o password
openssl rand -base64 24
```

## Errores comunes (con mensajes exactos)

| Mensaje | Causa | Fix |
|---|---|---|
| `bad decrypt` / `error:06065064` al descifrar | KDF/hash distinto al usado al cifrar, o passphrase equivocada | Añadir `-md md5`/`-sha256` según el origen; verificar passphrase |
| `no starttls protocol specified` | Olvidado `-starttls <proto>` en puerto no-TLS | `-starttls smtp` (o imap/pop/ldap/ftp…) |
| `connect:errno=111` / timeout | Puerto incorrecto o servicio caído | Comprobar puerto y que habla TLS (¿STARTTLS?) |
| `sslv3 alert handshake failure` | Versión/cipher forzado no soportado por el servidor | Probar sin `-tls1*`/`-cipher` y leer el cipher negociado |
| Navegador rechaza el cert autofirmado con "nombre no válido" | Falta SAN | Regenerar con `-addext subjectAltName=...` |
| `unable to get local issuer certificate` | El servidor no envió la intermediaria o falta el CA local | Pasar `-CAfile`/`-untrusted` con la cadena correcta |
| El comando se queda colgado | `s_client` espera la conexión | Terminar con `</dev/null` o `echo ... |` |
| `Unknown cipher` | Cipher list errónea o TLS1.3 (catálogo distinto) | `openssl ciphers` para nombres válidos; `ciphersuites` en TLS1.3 |

## Cheatsheet

```bash
openssl version -a                          # build y configuración
openssl genpkey -algorithm RSA -out k.pem -pkeyopt rsa_keygen_bits:4096
openssl req -x509 -newkey rsa:2048 -nodes -keyout k.pem -out c.pem -days 365 -subj "/CN=x" \
  -addext "subjectAltName=DNS:x,IP:127.0.0.1"
openssl req -new -key k.pem -out p.csr -subj "/CN=x"
openssl x509 -in c.pem -text -noout
openssl x509 -in c.pem -noout -dates -fingerprint -sha256
openssl verify -CAfile ca.pem -untrusted inter.pem cert.pem
echo | openssl s_client -connect host:443 -servername host -showcerts
echo | openssl s_client -starttls smtp -connect host:25
openssl ciphers -v 'HIGH' ; openssl ciphersuites
openssl enc -aes-256-cbc -salt -md sha256 -in f -out f.enc -pass pass:x
openssl dgst -sha256 archivo
openssl pkcs12 -export -out c.p12 -inkey k.pem -in c.pem
```

## Referencias

- Documentación oficial por versión: <https://www.openssl.org/docs/> (man1 de 3.x: <https://www.openssl.org/docs/man3.0/man1/>)
- Chef's table of openssl commands (referencia clásica): <https://www.sslshopper.com/article-most-common-openssl-commands.html>
- Guías relacionadas en el vault: [[SSLScan]] (barrido rápido), [[SSLyze Analizador SSL TLS]] (auditoría programática)
- Fase de reconocimiento: [[Reconocimiento y Escaneo de Redes y Web]]

## Relacionadas

- [[SSLScan]] — qué ciphers soporta el servidor, en lote
- [[SSLyze Analizador SSL TLS]] — auditoría TLS completa y CI
- [[Nmap Guía Completa]] — scripts `ssl-*` para el primer sondeo
- [[PadBuster - Herramienta de Ataque de Relleno Oracle]] — oráculos de relleno en el contexto criptográfico (véase Bleichenbacher/ROBOT)
