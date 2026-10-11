# Zsteg - Análisis LSB en PNG/BMP

> **Herramienta**: Detector y extractor de esteganografía LSB (Ruby, rápido, automatizado). Especializado en **PNG** y **BMP** sin pérdidas.
> **Repo**: https://github.com/zed-0xff/zsteg | `gem install zsteg`

---

## Instalación

```bash
# RubyGems (requiere Ruby >= 2.5)
gem install zsteg

# Verificar
zsteg --version
# zsteg 0.2.7

# Si error permisos: gem install --user-install zsteg
# Añadir ~/.gem/ruby/X.X.0/bin al PATH
```

---

## Qué Detecta (Auto-mágico)

| Método | Descripción | Formato |
|--------|-------------|---------|
| **LSB RGB** | Bit menos significativo en R, G, B (orden xy, yx, rand) | PNG, BMP |
| **LSB RGBA** | Incluye canal Alpha | PNG |
| **LSB por bit-plano** | Bits 1-4 (b1, b2, b3, b4) | PNG, BMP |
| **Canales individuales** | Solo R, solo G, solo B, solo A | PNG |
| **Orden de recorrido** | xy (filas), yx (columnas), random (semilla) | PNG, BMP |
| **XOR / Aditivo** | Payload XOR con patrón / suma modular | PNG |
| **Metadata chunks** | tEXt, zTXt, iTXt, comentarios | PNG |

---

## Uso Básico

```bash
# Análisis COMPLETO automático (recomendado inicio)
zsteg imagen.png

# Salida típica:
# b1,rgb,lsb,xy       .. text: "flag{lsb_easy}"
# b1,rgba,lsb,xy      .. text: "flag{lsb_easy}"
# b2,rgb,lsb,xy       .. file: data
# metadata,comment    .. text: "Comentario EXIF"
```

---

## Opciones Principales

| Flag | Descripción |
|------|-------------|
| `-a, --all` | Probar TODOS los métodos (lento, exhaustivo) |
| `-b, --bits BITS` | Qué bits probar: `1`, `1,2`, `1-4`, `lsb` (default: 1) |
| `-o, --order ORDER` | Orden: `xy`, `yx`, `rand`, `rand:N` (semilla) |
| `-c, --channel CH` | Canal: `rgb`, `rgba`, `r`, `g`, `b`, `a`, `gray` |
| `-m, --method MÉTODO` | `lsb`, `msb`, `lsbry`, `xor`, `add`, `sub` |
| `-E, --extract PATRÓN` | Extraer payload binario (ver sintaxis abajo) |
| `-v, --verbose` | Más detalles |
| `-q, --quiet` | Solo resultados positivos |
| `--png` | Forzar modo PNG |
| `--bmp` | Forzar modo BMP |

---

## Sintaxis de Extracción (`-E`)

```
zsteg -E "BITS,CANAL,MÉTODO,ORDEN" imagen.png > salida.bin

BITS:     b1, b2, b3, b4, b1-4, lsb, msb
CANAL:    rgb, rgba, r, g, b, a, gray, all
MÉTODO:   lsb, msb, lsbry, xor, add, sub
ORDEN:    xy, yx, rand, rand:SEMILLA
```

### Ejemplos Extracción

```bash
# 1. LSB bit 1, RGB, orden xy (más común)
zsteg -E "b1,rgb,lsb,xy" imagen.png > payload.bin

# 2. LSB bit 1-2, RGBA, orden yx
zsteg -E "b1-2,rgba,lsb,yx" imagen.png > payload.bin

# 3. Bit 3 solo canal Blue, random seed 12345
zsteg -E "b3,b,lsb,rand:12345" imagen.png > payload.bin

# 4. XOR con patrón (común en CTF)
zsteg -E "b1,rgb,xor,xy" imagen.png > payload.bin
```

---

## Flujo de Trabajo CTF Típico

```bash
#!/bin/bash
# zsteg_ctf.sh - Pipeline automatizado

IMG="$1"
OUTDIR="zsteg_out_$(basename "$IMG" .png)"
mkdir -p "$OUTDIR"

echo "[+] Análisis rápido..."
zsteg "$IMG" | tee "$OUTDIR/quick.txt"

echo "[+] Análisis exhaustivo (-a)..."
zsteg -a "$IMG" 2>/dev/null | tee "$OUTDIR/full.txt"

echo "[+] Extrayendo candidatos..."
# Extraer todos los hallazgos de texto
zsteg "$IMG" 2>/dev/null | grep "text:" | while read line; do
    method=$(echo "$line" | awk '{print $1}')
    echo "  Extrayendo: $method"
    zsteg -E "$method" "$IMG" > "$OUTDIR/${method}.bin" 2>/dev/null
    file "$OUTDIR/${method}.bin"
    strings "$OUTDIR/${method}.bin" | head -5
done

echo "[+] Revisar $OUTDIR/"
```

---

## Ejemplos Prácticos por Caso

### Caso 1: LSB Básico (CTF Principiante)
```bash
$ zsteg facil.png
b1,rgb,lsb,xy       .. text: "flag{easy_lsb_123}"

$ zsteg -E "b1,rgb,lsb,xy" facil.png
flag{easy_lsb_123}
```

### Caso 2: LSB en Canal Alpha (RGBA)
```bash
$ zsteg -a canal_alpha.png
b1,rgba,lsb,xy      .. text: "flag{alpha_channel}"
b1,rgb,lsb,xy       .. file: data (basura)

$ zsteg -E "b1,rgba,lsb,xy" canal_alpha.png
flag{alpha_channel}
```

### Caso 3: Bits Superiores (b2, b3, b4)
```bash
# Payload grande usa 2 bits por canal
$ zsteg -b 1,2 -a imagen.png
b1,rgb,lsb,xy       .. file: data
b2,rgb,lsb,xy       .. file: data
b1-2,rgb,lsb,xy     .. text: "flag{two_bits_per_channel}"

$ zsteg -E "b1-2,rgb,lsb,xy" imagen.png > payload.bin
```

### Caso 4: Orden Aleatorio con Semilla (Randomizado)
```bash
# Si el embedding usó PRNG con semilla conocida
$ zsteg -E "b1,rgb,lsb,rand:1337" imagen.png > payload.bin
# Probar semillas comunes: 0, 1, 123, 1337, 31337, 13371337, timestamp
for seed in 0 1 123 1337 31337 13371337 $(date +%s); do
    zsteg -E "b1,rgb,lsb,rand:$seed" imagen.png 2>/dev/null | strings | grep -q "flag" && echo "Seed: $seed"
done
```

### Caso 5: Metadata PNG (Chunks tEXt/zTXt/iTXt)
```bash
$ zsteg -a imagen.png
metadata,tEXt       .. text: "Author: CTF Player"
metadata,zTXt       .. text: "flag{hidden_in_ztxt}"
metadata,iTXt       .. text: "UTF-8: bandera{unicode}"

# Extraer chunk específico
zsteg -E "metadata,zTXt" imagen.png
# flag{hidden_in_ztxt}
```

### Caso 6: Archivo Binario Embebido (no texto)
```bash
$ zsteg -a sospechosa.png
b1,rgb,lsb,xy       .. file: data

$ zsteg -E "b1,rgb,lsb,xy" sospechosa.png > extraido.bin
$ file extraido.bin
# extraido.bin: ZIP archive data, at least v2.0 to extract
# o: gzip, PNG, JPEG, ELF, PDF, etc.

$ binwalk -e extraido.bin
$ unzip extraido.bin -d contenido/
```

---

## Análisis Avanzado

### Detectar Qué Bits Usan (Heurística)

```bash
# Script: probar bits 1-4 y ver cuál da texto legible
for bits in 1 2 3 4 "1-2" "1-3" "1-4" "2-4"; do
    echo "=== Testing bits: $bits ==="
    zsteg -b "$bits" -c rgb -m lsb -o xy imagen.png 2>/dev/null | grep -E "text:|file:" | head -3
done
```

### Comparar Cover Original vs Stego (Diferencial)

```bash
# Si tienes cover original limpio
zsteg cover_original.png > clean.txt
zsteg cover_stego.png > stego.txt

# Diferencia en hallazgos
diff -u clean.txt stego.txt | grep "^+b"
# Muestra solo nuevos hallazgos en stego
```

### Extraer Payload Parcial (Offset/Length)

```bash
# zsteg no tiene offset nativo → extraer todo y recortar
zsteg -E "b1,rgb,lsb,xy" imagen.png > full.bin
# Primeros 100 bytes
head -c 100 full.bin > header.bin
# Bytes 100-500
dd if=full.bin of=body.bin bs=1 skip=100 count=400
```

---

## Integración con Otras Herramientas

```bash
# 1. zsteg + binwalk (archivos embebidos)
zsteg -E "b1,rgb,lsb,xy" img.png | binwalk -e -

# 2. zsteg + strings (búsqueda rápida)
zsteg -E "b1,rgb,lsb,xy" img.png | strings | grep -i "flag\|password\|secret"

# 3. zsteg + steghide (JPEG con LSB? No, steghide es formato propio)
# Pero puedes convertir PNG→JPEG y probar steghide
convert img.png img.jpg && steghide info img.jpg

# 4. Pipeline completo CTF
zsteg -a img.png 2>/dev/null | \
grep "text:" | \
awk '{print $1}' | \
xargs -I{} sh -c 'zsteg -E {} img.png | strings | grep -i flag'
```

---

## Limitaciones

| Limitación | Detalle |
|------------|---------|
| **Solo PNG/BMP** | No JPEG (lossy), no WAV, no GIF animado |
| **Solo LSB espacial** | No DCT (JPEG), no DWT (JPEG2000), no DFT |
| **Sin passphrase** | No descifra payloads cifrados (solo extrae crudo) |
| **Sin estegoanálisis estadístico** | No chi-square, RS, SPA (solo extracción) |
| **Performance** | Lento en imágenes > 10MP con `-a` |

---

## Alternativas Complementarias

| Herramienta | Para qué |
|-------------|----------|
| **stegsolve** | GUI visual bit-planes, análisis manual |
| **stegonline.com** | Web, LSB interactivo, sin instalar |
| **pngcheck** | Chunks PNG detallados, CRC, estructura |
| **stegdetect** | JPEG (JSteg, OutGuess, F5) |
| **steghide** | JPEG/BMP/WAV con passphrase + cifrado |
| **Python/PIL** | LSB custom, offset, canales específicos |

---

## Cheatsheet Rápido

```bash
# 1. Análisis rápido (90% casos)
zsteg imagen.png

# 2. Exhaustivo (si 1 falla)
zsteg -a imagen.png

# 3. Solo metadata PNG
zsteg -E "metadata" imagen.png

# 4. Extraer hallazgo específico
zsteg -E "b1,rgb,lsb,xy" imagen.png > out.bin

# 5. Probar semillas random comunes
for s in 0 1 123 1337 31337; do zsteg -E "b1,rgb,lsb,rand:$s" img.png; done

# 6. Verificar archivo extraído
file out.bin && strings out.bin | head -20
```

---

## Casos Reales CTF (Referencia)

| Challenge | Técnica | Comando Ganador |
|-----------|---------|-----------------|
| PicoCTF "So Meta" | Metadata PNG | `zsteg -E "metadata" img.png` |
| HackTheBox "Stego" | LSB b1 RGB | `zsteg -E "b1,rgb,lsb,xy" img.png` |
| CTFlearn "Hidden" | LSB b1 RGBA | `zsteg -E "b1,rgba,lsb,xy" img.png` |
| RingZer0 "Stegano" | 2 bits/canal | `zsteg -E "b1-2,rgb,lsb,xy" img.png` |
| Custom "Random" | Seed 1337 | `zsteg -E "b1,rgb,lsb,rand:1337" img.png` |

---

*Etiquetas: #zsteg #lsb #png #bmp #ctf #esteganografia #ruby #extraccion #metadata*