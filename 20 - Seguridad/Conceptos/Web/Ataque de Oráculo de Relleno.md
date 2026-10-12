---
aliases: ["Padding Oracle Attack", "Ataque de Oráculo de Relleno", "CBC Oracle"]
tags:
  - sec/web
  - sec/crypto
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[JSON Web Tokens Explotación y Bypass]]"
  - "[[Abuso de APIs]]"
  - "[[Information disclosure]]"
  - "[[Condiciones de Carrera]]"
fuente: "https://portswigger.net/web-security/crypto/padding-oracle"
---

# Ataque de Oráculo de Relleno (Padding Oracle Attack)

Un ataque de oráculo de relleno (**Padding Oracle Attack**) es un ataque contra datos cifrados en modo CBC que permite al atacante **descifrar (y también fabricar) contenido cifrado sin conocer la clave**, usando como única información la reacción del servidor ante rellenos (padding) válidos o inválidos. Es uno de los ataques más elegantes de criptografía aplicada: con ~128 peticiones por byte (256/2, en promedio) recupera el texto plano completo.

Un **oráculo** hace referencia a una "indicación" que brinda al atacante información sobre si la acción que ejecuta es correcta o no. Imagina que estás jugando a un juego de mesa o de cartas con un niño: la cara se le ilumina con una gran sonrisa cuando cree que está a punto de hacer un buen movimiento. Eso es un oráculo. En tu caso, como oponente, puedes usar este oráculo para planear tu próximo movimiento según corresponda.

> [!info]
> En criptografía, un oráculo es una función o servicio que, al ser consultado con datos cifrados, revela información sobre la validez o propiedades de los datos descifrados, sin exponer la clave de cifrado. Esta "indicación" puede ser explícita (un mensaje de error) o implícita (diferencias de tiempo de respuesta).

El **relleno** (*padding*) es un término criptográfico específico. Los cifrados de bloque funcionan sobre bloques de tamaño fijo (8 bytes en DES, 16 en AES). Si los datos no llenan el último bloque, se completan automáticamente. Muchos esquemas exigen que el relleno esté **siempre** presente, incluso si la entrada original llenaba el bloque exacto — así el descifrado siempre puede quitarlo de forma no ambigua.

> [!tip]
> Los esquemas de relleno más comunes incluyen PKCS#7 (el más extendido), ANSI X9.23 e ISO 10126. Cada uno define reglas específicas sobre cómo se añaden los bytes de relleno y qué valores deben tener, lo que es crucial para su correcta eliminación tras el descifrado.

Al combinar ambos elementos, una implementación con un oráculo de relleno revela si los datos descifrados tienen un relleno válido. El oráculo podría ser tan sencillo como un error "Relleno no válido", o algo más sutil como un tiempo de respuesta considerablemente diferente.

> [!warning] Timing attacks como oráculo implícito
> Si el procesamiento de un relleno válido toma un tiempo diferente (aunque sea mínimo) al de uno inválido, un atacante puede explotar esa diferencia para inferir información byte a byte. Las redes modernas permiten detectar diferencias < 0.1 ms en remoto, lo que convierte a cualquier rama de error en un oráculo práctico en todos los lenguajes.

## Mecánica: cómo funciona CBC

Los cifrados de bloque tienen una **modo de operación** que determina la relación entre bloques. En **CBC** (*Cipher Block Chaining*) cada bloque de texto plano se XOR-ea con el bloque cifrado anterior antes de cifrarse; el primer bloque se XOR-ea con un **IV** (*Initialization Vector*) aleatorio. Así, cifrar dos veces el mismo mensaje con la misma clave produce salidas distintas.

**Cifrado** (bloques `P1..Pn`, clave `K`, `E` = cifrar):

```
C1 = E(P1 XOR IV)
C2 = E(P2 XOR C1)
C3 = E(P3 XOR C2)
...Cn = E(Pn XOR C(n-1))
```

![[Pasted image 20250826090918.png]]

**Descifrado** (`D` = descifrar; el XOR se aplica tras descifrar):

```
P1 = D(C1) XOR IV
P2 = D(C2) XOR C1
P3 = D(C3) XOR C2
...Pn = D(Cn) XOR C(n-1)
```

![[Pasted image 20250826091519.png]]

> [!info] La propiedad que el ataque explota
> El descifrado de `Cn` es **local**: `Pn = D(Cn) XOR C(n-1)`. `D(Cn)` depende solo del bloque y la clave — pero `C(n-1)` (o el IV) lo **controla el atacante** si puede modificar el texto cifrado. Manipulando el bloque anterior manipula el texto plano resultante, byte a byte, SIN tocar la clave.

### Relleno PKCS#7

Para que el texto plano encaje en bloques completos se añade relleno. En PKCS#7: si faltan `N` bytes, se añaden `N` bytes de valor `N`. Si el mensaje llena el bloque exacto, se añade un bloque entero de `0x08` (con bloques de 8):

![[Pasted image 20250826091746.png]]

En la imagen: la primera cadena no completa el bloque de 8 bytes; faltan dos → `0x02 0x02`. Reglas completas:

| Bytes que faltan | Relleno añadido |
|---|---|
| 2 | `02 02` |
| 1 | `01` |
| 0 (bloque lleno) | `08 08 08 08 08 08 08 08` (bloque entero) |

El descifrado correcto **quita el relleno validando**: el último byte `k` debe ser un valor plausible de relleno y los últimos `k` bytes deben coincidir. Si no coincide → error. **Ese error diferencial es el oráculo.**

### Condición de oráculo

Cuando una aplicación descifra y luego limpia el relleno, si un relleno inválido provoca un comportamiento detectable (error 500 vs 200, redirect vs render, respuesta más lenta, log distinto), tenemos un **oráculo de relleno**. Con él podemos descifrar cualquier bloque observado e incluso **partir de texto plano y fabricar un cifrado válido** — el escenario completo que convierte una cookie cifrada en sesión de administrador.

## El ataque, byte a byte (matemática paso a paso)

Notación: `C0` = IV, `C1..Cn` = bloques cifrados, `I` = valor **intermedio** de un bloque tras el descifrado pero **antes** del XOR: `I(Cn) = D(Cn)`. El texto plano es `Pn = I(Cn) XOR C(n-1)`.

Para recuperar `Pn`, el atacante envía pares `(C', Cn)` donde `C'` es un bloque que **fabrica** y controla byte a byte:

```
Pn_controlado = I(Cn) XOR C'
```

**Objetivo**: fabricar `C'` tal que `Pn_controlado` termine en un relleno válido que le diga algo sobre `I`.

### Paso 1 — recuperar el último byte `Pn[15]` (bloques de 16, posiciones 0..15)

Queremos que el último byte del texto plano fabricado sea `0x01` (relleno válido de 1 byte):

```
Pn[15]_controlado = I[15] XOR C'[15] = 0x01
⇒ I[15] = 0x01 XOR C'[15]
```

Se fuerza bruta `C'[15]` sobre los 256 valores posibles, enviando `(C', Cn)` al oráculo. Cuando responde **relleno válido**, se ha encontrado `I[15]`. Entonces:

```
Pn[15] real = I[15] XOR C(n-1)[15]
```

Un detalle: a veces la coincidencia es casual (un relleno `0x02 0x02` que también parece válido). La solución estándar: al encontrar un candidato, cambiar el byte anterior (`C'[14]`) para romper cualquier relleno más largo y reconfirmar.

### Paso 2 — bytes siguientes (de derecha a izquierda)

Para recuperar `Pn[j]`, se **construye** el relleno esperado de la derecha: se conocen ya `I[j+1..15]`, así que se rellena `C'` para que `Pn_controlado[j+1..15] = 0x02, 0x02` (por ejemplo, al ir por el byte 14) y se fuerza bruta `C'[j]` hasta que el oráculo acepte `...0x02 0x02`:

```
C'[k] = I[k] XOR desired_padding_byte   (para k > j, ya conocido)
C'[j] = candidato (fuerza bruta 0..255)
⇒ cuando el oráculo dice "válido": I[j] = C'[j] XOR 0x02
⇒ Pn[j] = I[j] XOR C(n-1)[j]
```

### Algoritmo completo (por cada bloque, hacia atrás)

| Paso | Acción | Peticiones (promedio) |
|---|---|---|
| 1 | Empezar por el último byte: forzar `C'[15]` hasta relleno `0x01` | 128 |
| 2 | `I[15]` conocido → `P[15] = I[15] XOR C(n-1)[15]` | 0 |
| 3 | Para `j = 14 … 0`: rellenar `C'` con el padding ya derivado, forzar `C'[j]` hasta válido | 128 × 15 |
| 4 | `I` completo del bloque ⇒ `P = I XOR bloque_anterior` | 0 |
| 5 | Repetir con el par `(C(n-2), C(n-1))` para descifrar bloques previos | — |

Total por bloque ≈ 128 × 16 ≈ **2048 peticiones**; por mensaje completo de `n` bloques ≈ 2048 × `n`. La operatoria con los valores intermedios `I`:

![[Pasted image 20250826092855.png]]

Como se ve en la imagen: `C15 = E7 XOR I15` — perdón, la relación correcta del descifrado es `P15 = D(C15) XOR E7` (donde `E7`/`C14` es el bloque anterior). Por conmutatividad del XOR: `I15 = C15 XOR P15`, y forzando `P15` vía el bloque anterior fabricado se despeja byte a byte. Lo esencial: **el XOR es conmutativo y el bloque anterior es tuyo**.

> [!info] De la decifrado a la fabricación (bit-flipping)
> La misma capacidad sirve para **modificar** texto plano: si quieres que el byte 5 de la cookie diga `X`, calculas `C'(5) = C(5) XOR valor_actual XOR X`. El descifrado reflejará tu cambio. Es el ataque de bit-flipping contra cookies firmadas-con-MAC-debajo-del-cifrado.

## Herramientas: PadBuster y amigos

La ejecución manual es tediosa; se automatiza:

| Herramienta | Notas |
|---|---|
| **PadBuster** (Perl) | El clásico. Descifra y **fabrica** ciphertexts nuevos. Necesita: URL, valor cifrado, tamaño de bloque y un oráculo (código de respuesta o diferencia) |
| **POODLE-TLS / DROWN / SWEET32** | Familias distintas que comparten la idea de "oráculo por relleno"; no confundirlas con el ataque genérico |
| **tlspoodle / custom scripts** | Oráculos a nivel TLS (fallback SSLv3) |
| Scripts propios (Python `padder`/`padbuster`) | Cuando el oráculo es no estándar (respuestas binarias, timing) |

Sintaxis de PadBuster:

```bash
padbuster <URL> <VALOR_CIFRADO> <TAMAÑO_BLOQUE> \
  -cookies 'auth=<VALOR_CIFRADO>' \
  [-plaintext 'user=admin'] \
  [-brute] [-prefix <IV_prependido>]
```

| Flag | Uso |
|---|---|
| posiciónles | URL, valor cifrado, block size (8 DES / 16 AES) |
| `-cookies` | cookie/s header para mantener sesión y definir dónde vive el cifrado |
| `-plaintext` | fabricar un texto plano nuevo (modo de **cifrado**, no solo descifrado) |
| `-encoding 2` | salida en base64 (1=hex, 2=base64, 3=raw, 6=Base64URL) |
| `-error` / `-prefix` | definir el oráculo alternativo / IV embebido como prefijo |
| `-brute` | fuerza bruta de IV cuando no se conoce |

## Detección

- Cualquier endpoint que refleje **errores diferenciados** ante tokens/cookies alterados: 500 vs 302, JSON con `"error":"bad padding"` vs `"error":"invalid user"`, respuesta vacía vs completa.
- Alterar un solo byte del ciphertext (XOR con `\xff`) y comparar respuestas: en CBC con relleno, ~1/256 de los cambios producirán "relleno válido" y el resto error — una distribución de respuestas bimodal es señal.
- Timing: respuestas con dos clusteres temporales distintos según byte modificado.
- Formatos típicos afectados: cookies ASP.NET (` ViewState` sin MAC), tokens de sesión cifrados en CBC+PKCS#7, parámetros `auth=`, JWT con `dir` simétrico mal implementado (raro), WebService fields.

## Impacto

1. **Descifrado total** de cualquier mensaje observado (incluido el tuyo — el IV y los bloques son "públicos" para el oráculo): sesiones, PII, tokens.
2. **Fabricación de mensajes**: cifrar `user=admin` sin conocer la clave → suplantación directa (como en la práctica de abajo).
3. **Bit-flipping**: alterar campos específicos (rol, precio) dentro del texto plano.
4. Cadenas: descifrar un token de reset (lectura) → administración (como el lab de PortSwigger con el parámetro `forgot-password`).

## Prevención (con código)

La única mitigación completa: **autenticar el texto cifrado ANTES de descifrar** (Encrypt-then-MAC), con comparación de tiempo constante:

```python
import hmac, hashlib, os
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes

KEY_ENC = os.urandom(32)
KEY_MAC = os.urandom(32)

def encrypt_then_mac(pt: bytes) -> bytes:
    iv = os.urandom(16)
    enc = Cipher(algorithms.AES(KEY_ENC), modes.CBC(iv)).encryptor()
    ct = iv + enc.update(pkcs7_pad(pt, 16)) + enc.finalize()
    mac = hmac.new(KEY_MAC, ct, hashlib.sha256).digest()   # MAC sobre IV+CT
    return mac + ct

def mac_then_decrypt(blob: bytes) -> bytes:
    mac, ct = blob[:32], blob[32:]
    expect = hmac.new(KEY_MAC, ct, hashlib.sha256).digest()
    if not hmac.compare_digest(mac, expect):      # tiempo constante, ANTES del descifrado
        raise ValueError("integridad")            # → nunca se llega al oráculo de relleno
    iv, body = ct[:16], ct[16:]
    dec = Cipher(algorithms.AES(KEY_ENC), modes.CBC(iv)).decryptor()
    return pkcs7_unpad(dec.update(body) + dec.finalize())
```

Reglas:

- **Encrypt-then-MAC** (HMAC-SHA256 con clave separada) o directamente **AEAD** (AES-GCM, ChaCha20-Poly1305) — los modos autenticados no tienen "relleno" que adivinar.
- Comparar MAC con `hmac.compare_digest` / `constant_time_equals`: una comparación `==` crea un **nuevo oráculo de timing**.
- Nunca exponer errores de descifrado diferenciados al cliente; loguea en servidor con mensaje genérico.
- Preferir AEAD sobre CBC+HMAC: elimina la clase entera de errores de implementación (orden MAC-then-encrypt, IV predecible, etc.).

## Errores comunes

**Atacante:**
1. Usar el tamaño de bloque equivocado (8 para AES; AES = 16): PadBuster fallará ruidosamente.
2. Ignorar el IV embebido/extra: si el IV viaja como prefijo del ciphertext, hay que pasárselo a la herramienta (`-prefix`) o el descifrado arranca mal.
3. Asumir que el oráculo es el código HTTP: a veces es el **tamaño** de respuesta, un redirect o el tiempo — definir el detector correcto.
4. En el modo `-plaintext`, olvidar que los bloques intermedios dependen de bloques anteriores: PadBuster maneja la cadena, pero entenderlo evita interpretar mal los resultados.

**Defensor (falsas soluciones):**
1. "Ciframos con AES": CBC sin integridad autenticada sigue vulnerable aunque la clave sea fuerte.
2. MAC pero **después** de validar el relleno (MAC-then-encrypt sin revisar antes): el oráculo de relleno sigue vivo.
3. MAC con comparación `==`: sigue existiendo un oráculo de tiempo.
4. Ocultar los mensajes de error: si la **falta de error** también es distinguible (redirect de éxito), el oráculo persiste.
5. IV aleatorio pero reutilizado / IV constante: rompe confidencialidad aunque no haga padding oracle.

## Cheatsheet

| Elemento | Valor |
|---|---|
| Bloque DES | 8 bytes |
| Bloque AES | 16 bytes |
| Peticiones por byte | ~128 (promedio), 256 (máximo) |
| Peticiones por bloque | ~2048 |
| PKCS#7 bloque completo | bloque entero `0x10` × 16 |
| Relación clave | `P = D(C) XOR bloque_anterior` |
| Bit-flip | `C' = C XOR P_actual XOR P_deseado` |
| Defensa | Encrypt-then-MAC / AES-GCM + `compare_digest` |

```bash
# Descifrar
padbuster http://target/index.php <COOKIE> 8 -cookies 'auth=<COOKIE>'
# Fabricar cookie válida con texto plano propio
padbuster http://target/index.php <COOKIE> 8 -cookies 'auth=<COOKIE>' -plaintext 'user=admin'
```

## Práctica con PadBuster

![[Pasted image 20250826090824.png]]

Esta es la interfaz web en la que vamos a practicar (máquina Vulnerable "Padding Oracle" de VulnHub, con cifrado aplicado en la cookie de sesión).

Entonces, vamos a registrarnos en la web:

![[Pasted image 20250826095747.png]]

![[Pasted image 20250826095800.png]]

Podemos ver que ya estamos registrados y que la sesión ha iniciado.

Ahora, el cifrado se aplica en la cookie, por lo que vamos a extraerla para analizarla:

![[Pasted image 20250826095940.png]]

Detrás de esta cookie tenemos texto claro y podemos intentar obtenerlo. Para ello, usaremos la herramienta `PadBuster`:

```bash
padbuster http://192.168.1.105/index.php H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC 8 -cookies 'auth=H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC'
```

![[Pasted image 20250826100401.png]]

Seleccionamos la opción recomendada (el detector de oráculo por defecto analiza las respuestas):

![[Pasted image 20250826100443.png]]

Como podemos ver, obtenemos el valor completo de la información. Como dato extra, solo podemos dividir los bloques por valores múltiplos de 8 (el tamaño del bloque del cifrado).

Ahora, la vulnerabilidad radica en que, una vez que el programa logró romper el cifrado, podemos recrear cookies válidas de otros usuarios para suplantar su información de la siguiente manera:

```bash
padbuster http://192.168.1.105/index.php H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC 8 -cookies 'auth=H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC' -plaintext 'user=admin'
```

![[Pasted image 20250826100908.png]]

Hemos generado el valor, como podemos observar. Con esto, podemos intentar usarlo para verificar si funciona:

![[Pasted image 20250826100944.png]]

¡Ya estamos como admin! El servidor solo validaba el relleno del CBC — sin MAC ni AEAD, fabricar `user=admin` era cuestión de peticiones.

## Laboratorios y referencias

- PortSwigger Academy — Padding oracle attack (lab online interactivo): https://portswigger.net/web-security/crypto/padding-oracle
- Pentester Lab – Padding Oracle (práctica propia con PadBuster): https://www.vulnhub.com/?q=padding+oracle
- Menezes, van Oorschot, Vanstone — *Handbook of Applied Cryptography*, cap. 7 (construcción del ataque)
- ASP.NET padding oracle (CVE-2010-3332 / MS10-070) — el incidente que popularizó la técnica

## Relacionadas

- [[JSON Web Tokens Explotación y Bypass]] — manipulación de tokens de sesión; si el token está cifrado (JWE) en CBC, este ataque aplica
- [[Information disclosure]] — el oráculo ES información filtrada por el canal de error
- [[Abuso de APIs]] — cookies/parámetros cifrados de APIs como superficie del ataque
- [[Condiciones de Carrera]] — otra clase de ataque donde el "comportamiento detectable" del servidor es el gancho
