---
aliases:
  - Esteganografía
  - Steganografía
  - Ocultamiento de Mensajes
  - Stego Fundamentos
tags:
  - sec/stego
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Estegoanálisis - Detección y ContraMedidas]]"
  - "[[Steghide - Uso Avanzado]]"
  - "[[Stegseek - Guía Rápida]]"
  - "[[Zsteg - Análisis LSB PNG]]"
  - "[[Wireshark y Tshark]]"
fuente: "https://en.wikipedia.org/wiki/Steganography"
---

# Esteganografía: Conceptos, Herramientas y Práctica

**Esteganografía** (*steganography*) es el arte y la ciencia de **ocultar la existencia misma de un mensaje** dentro de otro medio llamado portador (*cover*). A diferencia de la criptografía, que cifra el contenido y deja a la vista que hay un mensaje, la esteganografía hace que el portador parezca un archivo perfectamente normal: si nadie sospecha, nadie analiza. En la práctica ofensiva moderna se usa para exfiltración de datos, canales de comando y control (C2) y retos de CTF.

> [!info] La tríada de la ocultación
> - **Criptografía**: el mensaje se ve, pero no se entiende. Ataque: fuerza bruta / criptoanálisis.
> - **Esteganografía**: el mensaje no se ve. Ataque: estegoanálisis (detección estadística y extracción).
> - **Estegoanálisis**: rama defensora/forense que busca, detecta y extrae lo oculto (ver [[Estegoanálisis - Detección y ContraMedidas]]).
>
> La combinación fuerte es **cifrar primero, esteganografiar después**: AES-256 del payload y luego embedding. Así, aunque detecten el estego-objeto, el contenido sigue protegido.

## 1. Modelo matemático

```
Cover (portador) + Payload (mensaje) + Key (clave, opcional)  →  Stego-objeto
```

| Término | Significado |
|---|---|
| **Cover** | Archivo limpio: imagen, audio, vídeo, texto, paquete de red |
| **Payload / message** | Datos a ocultar: texto, archivo, shellcode, credenciales |
| **Key** | Parámetro secreto (passphrase, semilla PRNG) que gobierna embedding/extracción |
| **Stego-objeto** | Cover + payload embebido; debe ser indistinguible de una cover |
| **Capacity** | Máximo de bits que caben sin degradar el portador |
| **Imperceptibilidad** | Grado de invisibilidad visual/auditiva/estadística del cambio |
| **Robustez** | Sobrevivencia del payload a transformaciones (compresión, recorte, ruido) |

### El triángulo de compromiso

```
            Capacidad
           (bits ocultos)
                /\
               /  \
              /    \
   Robustez ---------- Imperceptibilidad
 (sobrevive a        (invisible visual y
  compresión,         estadísticamente)
  recorte, ruido)
```

- **Más capacidad** ⇒ más píxeles/bits modificados ⇒ **más ruido estadístico** ⇒ más detectable.
- **Más robustez** ⇒ redundancia/codificación (matrix embedding, corrección) ⇒ menos capacidad efectiva.
- **Más imperceptibilidad** ⇒ cambios mínimos y selectivos ⇒ **más frágil** ante transformaciones.

> [!tip] Regla práctica de CTF
> Si el archivo "se ve raro" o es sospechosamente grande para su tipo, empieza por capacidad: **capacidad usada ≈ tamaño del payload**. Un PNG de 800×600 con 3 canales de 8 bits admite teóricamente 800×600×3 = 1.440.000 bits = **180 KB** en LSB 1-bit; que un payload de 200 bytes quepa es trivial, pero que el histograma se deforme depende de cuántos bits se usen.

## 2. Breve historia

| Época | Técnica | Nota |
|---|---|---|
| Siglo V a.C. | Herodoto: tatuajes bajo vendajes, mensajes en tablillas de cera recubiertas | Origen documentado del concepto |
| Siglo XVI | Tratados de Trithemius (*Steganographia*, 1499) | Clasificación temprana: ocultación por cifrado + encubrimiento |
| Guerras mundiales | Microfilm, tinta invisible, letras de puntos en cartas | Uso militar/inteligencia |
| 1985 | Primeros trabajos digitales: *Comutational Steganography* | Fundamentos algorítmicos modernos |
| 2000s | LSB en imágenes digitales, steghide, JSteg, OutGuess, F5 | Explosión de herramientas open source |
| 2010s–hoy | DCT/DWT adaptativos, GANs/adversarial, canales de red (DNS, HTTP), malware (APT28, Stegoloader) | Ver [[Estegoanálisis - Detección y ContraMedidas]] §9 |

## 3. Taxonomía de técnicas

### 3.1 Dominio espacial (imágenes/audio)

| Técnica | Mecánica | Capacidad | Robustez | Detectabilidad |
|---|---|---|---|---|
| **LSB 1-bit** | Sustituye el bit menos significativo de cada canal (R,G,B/A) por bits del mensaje | ~12,5% del tamaño crudo (3/24 bits por píxel RGBA) | Muy baja: JPEG la destruye | Media (chi², RS, histogramas) |
| **LSB n-bits** | Usa los 2–4 bits menos significativos | 2–4× más | Muy baja | Alta (degrada visibilidad) |
| **LSB matching (±1)** | Si el LSB no coincide, suma/resta ±1 aleatorio en vez de forzar el bit | Igual | Baja | **Baja**: rompe la correlación par/impar que detecta χ² |
| **PVD** (*Pixel Value Differencing*) | Adapta la cantidad de bits según la diferencia entre píxeles vecinos (zonas planas → pocos bits) | Variable | Media | Baja (respeta textura) |
| **Embedding adaptativo (edge/texture)** | Solo modifica bordes y textura de alta varianza | Baja–media | Media | Muy baja (hoy el estándar) |
| **OutGuess/F5 (JPEG)** | Modifica coeficientes AC de la DCT preservando el histograma | Baja–media | **Alta** (sobrevive recompresión) | Baja si usa matrix embedding |

**Mecánica LSB 1-bit (24 bpp RGB):**

```
Píxel original:   R=11010101  G=01101100  B=10011010
Mensaje (bits):    1            0            1
Píxel modificado: R=11010101  G=01101100  B=10011011   ← solo la LSB de B cambió
```

Cada canal aporta 1 bit por píxel: 3 canales × 1 bit = 3 bits por píxel; a 8 bits/byte, un byte de mensaje consume 8/3 ≈ 2,67 píxeles.

### 3.2 Dominio de frecuencia / espectral

| Transformada | Dónde vive | Formato asociado | Comentario |
|---|---|---|---|
| **DCT** (Discrete Cosine Transform) | Coeficientes AC de bloques 8×8 | JPEG, MPEG | Base de JSteg, OutGuess, F5; resistente a compresión lossy |
| **DWT** (Discrete Wavelet Transform) | Bandas de aproximación/detalle | JPEG2000, WAV | Multi-resolución, buena robustez |
| **DFT** (Discrete Fourier Transform) | Espectro de frecuencias | Análisis genérico | Frecuente en audio (phase coding, LSB en frecuencias seleccionadas) |

> [!info] Por qué el dominio frecuencial es más robusto
> La compresión JPEG descarta información de alta frecuencia; si el payload vive en coeficientes AC medios, **sobrevive** al reescalonado/recompresión. El precio: cada cambio afecta muchos píxeles tras la transformada inversa, lo que exige algoritmos más cuidadosos (matrix embedding, permutación con clave).

### 3.3 Formato y metadatos

| Método | Mecánica | Herramienta típica | Detección |
|---|---|---|---|
| **Appending (EOF)** | Datos tras el marcador final (`FFD9` JPEG, `IEND` PNG) | `cat secreto.zip >> foto.jpg` | `binwalk`, `xxd`, lectura de estructura |
| **Chunks PNG** | `tEXt`, `zTXt`, `iTXt`, `zlib`, chunks privados `xxxx` | `exiftool -Comment=...`, scripts | `pngcheck -v`, `zsteg` (metadata) |
| **Segmentos JPEG** | `APPn` (EXIF=`APP1`), `COM` (segmento de comentario) | `exiftool` | `exiftool`, `jpegdump.py` |
| **Polyglots** | Archivo válido en ≥2 formatos (JPEG+ZIP, PDF+JS) | concatenación cuidadosa / scripts | `unzip -l archivo.jpg`, firmas cruzadas |
| **Cabeceras falsas / padding** | Extensiones engañosas (GIF89a como cabecera de un ejecutable) | edición manual | `file`, `xxd` |

### 3.4 Texto y Unicode

| Técnica | Mecánica | Riesgo |
|---|---|---|
| **Zero-width chars** | Inserción de U+200B/U+200C/U+200D/U+FEFF entre caracteres visibles | `stegcloak`, detectable con `xxd`/scripts |
| **Homoglifos / synonym encoding** | Sustitución de palabras o caracteres visualmente idénticos | Difícil de detectar sin original |
| **Whitespace encoding** | Espacios/tabuladores al final de líneas codificando bits | Visible con `cat -A` |

### 3.5 Red (canales encubiertos)

| Canal | Mecánica | Detección |
|---|---|---|
| **DNS** (tunnels: DNSCat, Iodine) | Subdominios que codifican datos en consultas A/TXT | Entropía de consultas, volumen, dominios largos |
| **HTTP headers / User-Agent / cookies** | Datos en campos que la app ignora | Inspección profunda (DPI), WAF |
| **Timing / inter-packet delay** | Modulación en la latencia entre paquetes | Análisis estadístico de timings |
| **ICMP payload / covert channels** | Datos dentro del echo-request legítimo | `tcpdump`/[[Wireshark y Tshark]]: tamaños ICMP anómalos |
| **Stego en imágenes servidas por web** | C2 que descarga comandos desde imágenes LSB en GitHub/CDN |_hashes de imágenes, anomalías estadísticas |

## 4. Catálogo de herramientas

### 4.1 Embedding / extracción

| Herramienta | Lenguaje | Formatos | Función clave | Nota |
|---|---|---|---|---|
| **[[Steghide - Uso Avanzado\|steghide]]** | C | JPEG, BMP, WAV, AU | `embed` / `extract` / `info` con passphrase + AES-256 | Estándar de CTF; **no soporta PNG** |
| **[[Stegseek - Guía Rápida\|stegseek]]** | Go | (steghide) | Fuerza bruta de passphrase a ~1M intentos/s | Ver [[Stegseek - Guía Rápida]] |
| **[[Zsteg - Análisis LSB PNG\|zsteg]]** | Ruby | PNG, BMP | Detecta/extrae LSB por canal, bit, orden; metadata | Ver [[Zsteg - Análisis LSB PNG]] |
| **stegsolve** | Java (GUI) | imágenes | Explorador de bit-planes, canales, XOR, concatenación | Manual, imprescindible para análisis visual |
| **stegonline** | Web | PNG/JPEG | LSB interactivo, canales, XOR, extracción | Sin instalación |
| **openstego** | Java | PNG, JPEG | Múltiples algoritmos + plugins | Deja firmas propias |
| **outguess / jsteg / f5** | C | JPEG | Estego en coeficientes DCT | Clásicos académicos |
| **stegcloak** | JS/CLI | texto | Zero-width en Unicode | Texto, no binario |

### 4.2 Análisis y apoyo

| Herramienta | Función | Uso típico |
|---|---|---|
| **exiftool** | Leer/escribir metadatos EXIF/IPTC/XMP, comentarios | `exiftool img.jpg` |
| **binwalk** | Firma de archivos embebidos + entropía por offset | `binwalk -e img.jpg` |
| **pngcheck** | Validación y volcado de chunks PNG + CRC | `pngcheck -v img.png` |
| **foremost / scalpel** | File carving por firmas | Recuperar archivos de imágenes/dumps |
| **strings / xxd / hexdump** | Cadenas e inspección hex | Primer barrido de cualquier archivo |
| **stegdetect / stegspy** | Firmas de algoritmos JPEG conocidos | Detección (ver [[Estegoanálisis - Detección y ContraMedidas]]) |
| **ent** | Entropía de Shannon por byte | Distinguir cifrado/estego de contenido natural |

## 5. Metodología de embedding (lado atacante)

### 5.1 Preparar el payload

```bash
# 1) Comprimir (payload binario comprime mal, texto mucho)
tar -czf datos.tar.gz carpeta_secreta/

# 2) Cifrar SIEMPRE antes de embeber (AES-256 + PBKDF2)
openssl enc -aes-256-cbc -salt -pbkdf2 -in datos.tar.gz -out payload.enc
# passphrase: definirla fuera del comando o con -pass stdin para no dejarla en historial
```

### 5.2 Elegir el portador (*cover*)

| Criterio | Buena cover | Mala cover |
|---|---|---|
| Textura | Foto natural, ruido de sensor, césped/agua/arena | Fondos sólidos, capturas de pantalla, logos, PNG con pocas variaciones |
| Formato | PNG/BMP/WAV/TIFF (sin pérdidas) si usas LSB; JPEG si usas steghide/DCT | GIF de 256 colores con LSB |
| Tamaño | `capacidad ≥ tamaño(payload) × 1,2` | Cover minúscula con payload enorme |
| Historial | Cover **nunca publicada antes** (evita análisis diferencial) | Cover reutilizada de otra operación |
| Tamaño final | Delta payload+overhead ≈ decenas de KB | 50 KB → 50 MB (alerta inmediata) |

### 5.3 Embedding con steghide (referencia)

```bash
steghide embed -cf foto.jpg -ef payload.enc -p "MiPassphraseFuerte123!" -z 9
# -cf  cover file (portador)
# -ef  embed file (payload a ocultar)
# -p   passphrase (si se omite, la pide en modo interactivo)
# -z   compresión zlib 1-9 antes de cifrar (9 = máxima)
# -e   algoritmo de cifrado (rijndael-256 por defecto; serpent-256, twofish-256…)
# -K   usar keyfile binario de 32 bytes en lugar de passphrase
# -N   no guardar el nombre del archivo oculto (gana bytes, pierde metadato)
# -f   sobrescribir la cover
```

Detalle completo de flags, capacidad y errores: [[Steghide - Uso Avanzado]].

### 5.4 Embedding LSB en PNG (zsteg / Python)

```bash
# zsteg embebe (LSB simple) — cubre los casos básicos de CTF
zsteg -E "payload.txt" cover.png stego.png
```

```python
# LSB 1-bit RGB mínimo y correcto (delimiter NUL al final)
from PIL import Image
img = Image.open("cover.png"); px = img.load()
bits = "".join(f"{b:08b}" for b in open("payload.enc", "rb").read()) + "00000000"
i = 0
for y in range(img.height):
    for x in range(img.width):
        if i >= len(bits): break
        r, g, b = px[x, y]
        for setter in ("r", "g", "b"):
            if i < len(bits):
                v = {"r": r, "g": g, "b": b}[setter]
                v = (v & 0xFE) | int(bits[i]); i += 1
                if setter == "r": r = v
                elif setter == "g": g = v
                else: b = v
        px[x, y] = (r, g, b)
    if i >= len(bits): break
img.save("stego.png")
```

## 6. Metodología de extracción (lado defensor/CTF)

Orden recomendado — cada paso descarta hipótesis:

| # | Paso | Comando | Qué busca |
|---|---|---|---|
| 1 | Identificar | `file img.jpg && exiftool img.jpg` | Tipo real, dimensiones, comentario, software que lo creó |
| 2 | Cadenas | `strings -n 6 img.jpg \| grep -iE 'flag\|pass\|key\|secret'` | Flags/trampas visibles, cabeceras de archivos |
| 3 | Estructura | `binwalk -e img.jpg` · `pngcheck -v img.png` | Archivos embebidos, chunks anómalos, CRC rotos |
| 4 | LSB (PNG/BMP) | `zsteg -a img.png` | Bit-planes, metadata, XOR/add |
| 5 | Steghide (JPEG/BMP/WAV/AU) | `steghide info img.jpg` | ¿Datos steghide? ¿con passphrase? |
| 6 | Passphrase débil | `stegseek --crack img.jpg rockyou.txt` | Fuerza bruta de la passphrase |
| 7 | Visual | stegsolve / stegonline: bit-planes, canales, XOR, diferencia | Patrones visibles en LSB |
| 8 | Estadística | `stegdetect`, χ², entropía | Confirmar presencia (ver [[Estegoanálisis - Detección y ContraMedidas]]) |
| 9 | Recursión | `file extraído && strings extraído` | El payload suele ser otro contenedor |

## 7. Ejemplos prácticos de CTF

### Ejemplo 1 — Flag en LSB de PNG

```bash
file flag.png
# PNG image data, 800 x 600, 8-bit/color RGB, non-interlaced
zsteg flag.png
# b1,rgb,lsb,xy  .. text: "flag{st3g0_1s_fun_123}"
zsteg -E "b1,rgb,lsb,xy" flag.png > flag.txt && cat flag.txt
# flag{st3g0_1s_fun_123}
```

### Ejemplo 2 — JPEG con steghide + passphrase débil

```bash
steghide info desafio.jpg
# embedded file: 1024 bytes, encrypted: rijndael-256  → sí es steghide
stegseek --crack desafio.jpg /usr/share/wordlists/rockyou.txt
# [+] Found passphrase: "secret123"   /   [+] Extracting to "desafio.jpg.out"
cat desafio.jpg.out        # flag{...} o binario → repetir paso 9 de la tabla
```

### Ejemplo 3 — Polyglot JPEG+ZIP

```bash
cat foto.jpg secreto.zip > polyglot.jpg
file polyglot.jpg      # JPEG image data  (sigue siendo JPEG válido)
unzip -l polyglot.jpg  # …y también ZIP válido
unzip polyglot.jpg -d extraido/
```

### Ejemplo 4 — C2 vía imagen (conceptual, laboratorio)

```bash
# Atacante: comando cifrado dentro de una imagen del servidor web
echo "id; hostname" | openssl enc -aes-256-cbc -pbkdf2 -k "ClaveC2" > cmd.enc
steghide embed -cf banner.jpg -ef cmd.enc -p "ClaveC2" -z 9 -f
# Víctima (implant): descargar, extraer, ejecutar
steghide extract -sf banner.jpg -xf cmd.enc -p "ClaveC2"
openssl enc -d -aes-256-cbc -pbkdf2 -k "ClaveC2" -in cmd.enc | sh
```

> [!warning] Este flujo es documentación de laboratorio
> El ejemplo anterior describe el patrón que usan campañas reales (APT28, Stegoloader). Reproduce **solo** en laboratorios propios o con autorización escrita; la sección 9 cubre cómo detectarlo como defensor.

## 8. OPSEC y buenas prácticas

| Regla | Por qué |
|---|---|
| Cifrar **antes** de esteganografiar | Defensa en profundidad: detección ≠ lectura |
| Cover única por operación | El análisis diferencial entre covers gemelas delata el embedding |
| Covers de alta entropía natural | Ruido de sensor, fotos RAW; nada de capturas ni gráficos planos |
| Usar ≤ 10–20% de la capacidad | Menos bits ⇒ menos deformación estadística |
| Limpiar metadatos de la cover | `exiftool -all= -overwrite_original cover.jpg` |
| Formato sin pérdidas para LSB | PNG/BMP/WAV; JPEG destruye el LSB en cada recompressão |
| Verificar la extracción **antes** de transmitir | Evita payloads corruptos silenciosos |
| Evitar firmas de herramienta | steghide/openstego dejan artefactos estructurales detectables |
| Passphrase larga y aleatoria | Ver [[Stegseek - Guía Rápida]]: rockyou rompe passphrases débiles en segundos |

## 9. Detección: el lado contrario

Resumen; teoría completa en [[Estegoanálisis - Detección y ContraMedidas]]:

- **Visual**: bit-planes con stegsolve, diferencia cover/stego (`convert a.png b.png -compose difference -composite diff.png`).
- **Estadístico**: χ² (uniformidad par/impar), RS analysis, sample-pair analysis, entropía por bloques.
- **Estructural**: `pngcheck` (chunks, CRC), segmentos `APPn`/`COM` anómalos, datos tras EOI.
- **Por firma**: `steghide info`, `stegdetect`, `stegspy`.
- **ML**: SRNet/Xu-Net (~95% en JPEG con datasets BOSSBase/ALASKA2), costoso de operar.

## 10. Errores comunes

1. **Usar estego en JPEG con LSB planilla**: la compresión lossy elimina la LSB en la siguiente guarda/recompresión. LSB ⇒ PNG/BMP/WAV; JPEG ⇒ steghide/F5/outguess.
2. **Asumir que `steghide` acepta PNG**: no; formatos soportados: JPEG, BMP, WAV, AU. `steghide info x.png` → *"unsupported format"*.
3. **Omitir el cifrado** y confiar solo en la ocultación: estegoanálisis + extracción = payload en claro.
4. **Payload mayor que la capacidad**: steghide falla con *"cover file too small"*; los scripts LSB silenciosamente **truncan** el mensaje (agrega un delimitador NUL y verifica la extracción).
5. **Reutilizar covers publicadas**: análisis diferencial con la cover original = extracción trivial.
6. **Dejar la passphrase en el historial** (`-p 'xxx'`): usar `-p "$VAR"` con variable de entorno o modo interactivo.
7. **Confundir ocultación con integridad**: sin checksum o con `-N`, la corrupción no avisa; verificar con `file`/`strings` siempre.
8. **Buscar solo `strings`**: muchos payloads son binarios (ZIP/ELF); `strings` no muestra cabeceras ZIP dentro de ruido.

## 11. Cheatsheet

```bash
# ── Identificación ──
file IMG; exiftool IMG; strings -n 6 IMG | grep -iE 'flag|pass|key'
binwalk -e IMG; pngcheck -v IMG.png          # PNG

# ── Embed (lado atacante) ──
openssl enc -aes-256-cbc -salt -pbkdf2 -in payload.tar.gz -out payload.enc
steghide embed -cf foto.jpg -ef payload.enc -p "$PASS" -z 9      # JPEG/BMP/WAV/AU
zsteg -E "payload.txt" cover.png stego.png                        # PNG LSB

# ── Extract (lado CTF/forense) ──
zsteg -a img.png                                  # análisis LSB completo
zsteg -E "b1,rgb,lsb,xy" img.png > out.bin        # extracción dirigida
steghide info img.jpg                             # ¿steghide?
stegseek --crack img.jpg ~/wordlists/rockyou.txt  # passphrase
steghide extract -sf img.jpg -xf out.bin -p "$PASS"

# ── Detección ──
stegdetect sospechoso.jpg; ent sospechoso.jpg
```

**Checklist de CTF**: `file` → `exiftool` → `strings` → `binwalk` → `zsteg` (PNG/BMP) → `steghide info` (JPG/BMP/WAV) → `stegseek` → stegsolve visual → χ²/estego → extraer y repetir recursivamente.

## 12. Referencias

- Steghide (documentación oficial): <http://steghide.sourceforge.net/documentation/>
- Stegseek (repo): <https://github.com/RickdeJager/stegseek>
- zsteg (repo): <https://github.com/zed-0xff/zsteg>
- Binwalk: <https://github.com/ReFirmLabs/binwalk>
- Fridrich, J. et al. — *Steganalysis of LSB Encoding in Color Images* (2001)
- Ker, A. — *Steganalysis of LSB Matching in Grayscale Images* (2005)
- Pevný, T. et al. — *Steganalysis by Subtractive Pixel Adjacency Matrix* (2010)
- CTF practice: <https://cryptohack.org/> · <https://www.hackthebox.com/>
- Wordlists: `/usr/share/wordlists/rockyou.txt.gz` · <https://github.com/danielmiessler/SecLists>

## Relacionadas

- [[Estegoanálisis - Detección y ContraMedidas]] — teoría de detección (χ², RS, visual) y contramedidas
- [[Steghide - Uso Avanzado]] — flags completos, capacidad, passphrase modes
- [[Stegseek - Guía Rápida]] — cracking de passphrases steghide con wordlists
- [[Zsteg - Análisis LSB PNG]] — extracción LSB en PNG/BMP y cadenas CTF
- [[Wireshark y Tshark]] — canales encubiertos y análisis de tráfico
- [[GHex Editor Hexadecimal]] — inspección hex para polyglots y appends
