---
tags:
  - ciberseguridad
  - esteganografia
  - zsteg
tipo: concepto
estado: terminado
---

# Zsteg — Análisis LSB en PNG/BMP

**zsteg** (Ruby, `gem install zsteg`) es el detector y extractor de esteganografía **LSB** por excelencia para **PNG y BMP**: prueba automáticamente decenas de combinaciones de canal, bit-plane, orden de recorrido y método, y reporta los candidatos que contienen texto o firmas de archivos. Es la primera herramienta que se corre ante cualquier imagen sin pérdidas en un CTF.

> [!info] Qué es exactamente la LSB y por qué se puede extraer sin passphrase
> Cada canal de un píxel PNG de 8 bits (`R G B [A]`) tiene 8 bits de información; el **bit menos significativo (LSB)** aporta 1/255 de intensidad — invisible a simple vista. Si alguien ocultó un mensaje, lo escribió reemplazando esas LSB por los bits del payload, recorriendo los píxeles en algún **orden** (filas `xy`, columnas `yx` o PRNG con semilla). zsteg no "descifra": **lee exactamente esos bits en el mismo orden** y devuelve los bytes. Por eso:
> - Funciona solo en formatos **sin pérdidas** (PNG/BMP): JPEG destruiría las LSB.
> - No hace falta passphrase **si no la usaron**; si el payload estaba cifrado, obtendrás basura → recurrir a [[Steghide - Uso Avanzado]]/[[Stegseek - Guía Rápida]] o a otro esquema.

## Instalación

```bash
gem install zsteg                # requiere Ruby >= 2.5
zsteg --version                  # zsteg 0.2.7
# sin permisos:
gem install --user-install zsteg
# añadir ~/.gem/ruby/X.X.0/bin al PATH
```

## Mecánica: los cuatro ejes de un ataque LSB

Todo método LSB queda definido por cuatro coordenadas; zsteg recorre el producto cartesiano:

| Eje | Valores | Significado |
|---|---|---|
| **Bits** | `b1`…`b4`, `b1-4`, `lsb`, `msb` | Qué bit-planes se leen (1 = LSB real) |
| **Canal** | `rgb`, `rgba`, `r`, `g`, `b`, `a`, `gray` | Dónde se mira |
| **Método** | `lsb`, `msb`, `lsbry`, `xor`, `add`, `sub` | Cómo se combinan los bits (directo, aleatorio, XOR con patrón, suma/resta modular) |
| **Orden** | `xy`, `yx`, `rand`, `rand:SEED` | Recorrido de los píxeles (filas, columnas, PRNG con semilla) |

Con eso se entiende la salida: `b1,rgb,lsb,xy` = *"leyó el bit 1 de los canales R,G,B recorriendo en filas"*.

## Tabla de flags

| Flag | Descripción |
|---|---|
| *(sin flags)* | Análisis rápido de los métodos por defecto — 90% de los casos CTF |
| `-a, --all` | Prueba **todos** los métodos (lento en imágenes grandes) |
| `-b, --bits LIST` | Bits a probar: `1`, `1,2`, `1-4`, `lsb` (defecto `1`) |
| `-o, --order ORD` | `xy`, `yx`, `rand`, `rand:N` (semilla) |
| `-c, --channel CH` | `rgb`, `rgba`, `r`, `g`, `b`, `a`, `gray` |
| `-m, --method M` | `lsb`, `msb`, `lsbry`, `xor`, `add`, `sub` |
| `-E, --extract SPEC` | Extraer el payload binario de una combinación concreta |
| `-v, --verbose` | Más detalle |
| `-q, --quiet` | Solo resultados positivos |
| `--png` / `--bmp` | Forzar el analizador de formato |

### Sintaxis de `-E`

```
zsteg -E "BITS,CANAL,MÉTODO,ORDEN" imagen.png > salida.bin
# ejemplos:
zsteg -E "b1,rgb,lsb,xy"      imagen.png > payload.bin   # el más común
zsteg -E "b1-2,rgba,lsb,yx"   imagen.png > payload.bin
zsteg -E "b3,b,lsb,rand:12345" imagen.png > payload.bin
zsteg -E "b1,rgb,xor,xy"      imagen.png > payload.bin   # XOR con patrón (CTF)
zsteg -E "metadata,zTXt"      imagen.png                 # chunk de texto PNG
```

## Interpretar la salida

```
$ zsteg imagen.png
b1,rgb,lsb,xy    .. text: "flag{lsb_easy}"      ← texto legible: hallazgo directo
b1,rgba,lsb,xy   .. text: "flag{lsb_easy}"      ← mismo payload con canal alfa
b2,rgb,lsb,xy    .. file: data                  ← payload binario: extraer y analizar
metadata,tEXt    .. text: "Author: CTF Player"  ← metadato PNG
```

| Resultado | Qué significa | Acción |
|---|---|---|
| `text: "..."` con flag/URL | Payload en claro | Copiar; si es binario raro, extraer igual |
| `file: data` | Cabecera de archivo (ZIP/PNG/ELF/PDF…) | `-E` → `file` → `binwalk` recursivo |
| `text: basura` (caracteres no imprimibles) | Payload **cifrado** u otro orden/método | Probar `-a`, otros bits, seeds |
| Solo aparece con `-a` | Método no estándar (xor/add/rand) | Iterar sobre esas combinaciones |
| Nada con `b1` pero sí con `b1-2` | Embedding de 2 bits/canal | Usar `b1-2` en `-E` |
| `metadata,*` | Chunk PNG con texto | Extraer; a veces la flag está ahí sin LSB |

> [!warning] Falsos positivos de zsteg
> Con `-a` aparecerán decenas de líneas `text:` basura: cualquier secuencia de bits "parece" texto. **Filtra por legibilidad** (`grep -aE 'flag|ctf|http|[a-z]{8,}'`) y valida con `file` antes de dar nada por hallazgo. El orden correcto es: extraer → inspeccionar → descartar con evidencia.

## Cadena CTF típica (zsteg → flag)

```bash
# 1) Identificación
file a.png
# PNG image data, 800 x 600, 8-bit/color RGB, non-interlaced

# 2) Análisis rápido
zsteg a.png
# b1,rgb,lsb,xy  .. text: "flag{st3g0_1s_fun_123}"

# 3) Extracción (o copy directo si el text ya es la flag)
zsteg -E "b1,rgb,lsb,xy" a.png > out.bin
cat out.bin                     # flag{...}

# ── Variante: payload binario ──
zsteg -E "b1,rgb,lsb,xy" a.png > out.bin
file out.bin                    # ZIP archive data …
unzip out.bin -d extraido/      # o: binwalk -e out.bin
strings -n 6 extraido/* | grep -i flag

# ── Variante: cadena larga en base64 ──
zsteg -E "b1,rgb,lsb,xy" a.png | grep -oE '[A-Za-z0-9+/=]{20,}' | base64 -d
```

**Cadena completa típica**: `zsteg` → `-E` → `file`/`strings` → `base64 -d`/`unzip` → flag. Ver también [[Esteganografía - Conceptos, Herramientas y Práctica]] §7.

## Casos prácticos

### Caso 1 — LSB básico (principiante)

```bash
$ zsteg facil.png
b1,rgb,lsb,xy  .. text: "flag{easy_lsb_123}"
$ zsteg -E "b1,rgb,lsb,xy" facil.png ; echo
flag{easy_lsb_123}
```

### Caso 2 — Canal Alpha

```bash
$ zsteg -a canal_alpha.png
b1,rgba,lsb,xy  .. text: "flag{alpha_channel}"
$ zsteg -E "b1,rgba,lsb,xy" canal_alpha.png
flag{alpha_channel}
```

### Caso 3 — Dos bits por canal

```bash
$ zsteg -b 1,2 -a imagen.png
b1-2,rgb,lsb,xy  .. text: "flag{two_bits_per_channel}"
$ zsteg -E "b1-2,rgb,lsb,xy" imagen.png
flag{two_bits_per_channel}
```

### Caso 4 — Orden aleatorio con semilla

```bash
# Si el embebido usó un PRNG con semilla conocida:
for seed in 0 1 123 1337 31337 13371337 $(date +%s); do
    zsteg -E "b1,rgb,lsb,rand:$seed" imagen.png 2>/dev/null | strings | grep -q flag \
        && echo "[+] Seed: $seed"
done
```

### Caso 5 — Metadatos PNG (tEXt/zTXt/iTXt)

```bash
$ zsteg -a imagen.png
metadata,zTXt  .. text: "flag{hidden_in_ztxt}"
$ zsteg -E "metadata,zTXt" imagen.png
flag{hidden_in_ztxt}
# alternativa sin LSB:
exiftool imagen.png ; pngcheck -v imagen.png
```

### Caso 6 — Archivo embebido (no texto)

```bash
$ zsteg -a sospechosa.png
b1,rgb,lsb,xy  .. file: data
$ zsteg -E "b1,rgb,lsb,xy" sospechosa.png > extraido.bin
$ file extraido.bin
extraido.bin: ZIP archive data, at least v2.0 to extract
$ binwalk -e extraido.bin && unzip extraido.bin -d contenido/
```

## Técnicas avanzadas

### Encontrar qué bits usa el embebido

```bash
for bits in 1 2 3 4 "1-2" "1-3" "1-4" "2-4"; do
  echo "=== bits: $bits ==="
  zsteg -b "$bits" -c rgb -m lsb -o xy imagen.png 2>/dev/null | grep -E "text:|file:" | head -3
done
```

### Diferencia entre cover limpia y estego

```bash
zsteg cover_original.png > clean.txt
zsteg cover_stego.png     > stego.txt
diff -u clean.txt stego.txt | grep '^+b'     # solo los hallazgos nuevos
```

### Pipeline automatizado

```bash
#!/bin/bash
# zsteg_ctf.sh — análisis + extracción en lote
IMG="$1"; OUT="zsteg_out_$(basename "$IMG" .png)"; mkdir -p "$OUT"
zsteg "$IMG"   | tee "$OUT/quick.txt"
zsteg -a "$IMG" 2>/dev/null | tee "$OUT/full.txt"
grep -E "text:|file:" "$OUT/full.txt" | awk '{print $1}' | while read -r m; do
  zsteg -E "$m" "$IMG" > "$OUT/${m//,/_}.bin" 2>/dev/null
  echo "── $m"; file "$OUT/${m//,/_}.bin"; strings -n 6 "$OUT/${m//,/_}.bin" | head -5
done
ls -la "$OUT/"
```

### Recortar payload (zsteg no tiene offset)

```bash
zsteg -E "b1,rgb,lsb,xy" img.png > full.bin
head -c 100 full.bin > header.bin
dd if=full.bin of=body.bin bs=1 skip=100 count=400
```

## Integración con otras herramientas

```bash
zsteg -E "b1,rgb,lsb,xy" img.png | binwalk -e -                 # contenedores
zsteg -E "b1,rgb,lsb,xy" img.png | strings | grep -iE 'flag|pass|secret'
convert img.png img.jpg && steghide info img.jpg                 # si el reto "cambia" de formato
zsteg -a img.png 2>/dev/null | grep "text:" | awk '{print $1}' | \
  xargs -I{} sh -c 'zsteg -E {} img.png | strings | grep -i flag'
```

| Herramienta | Papel complementario |
|---|---|
| **stegsolve** | Bit-planes visuales, XOR, concatenación (lo que zsteg no "ve") |
| **pngcheck** | Chunks y CRC sin intentar extracciones |
| **steghide/stegseek** | Ruta distinta: JPEG/BMP/WAV con passphrase |
| **Python/PIL** | Offset, canal o recorrido personalizados |
| **stegonline** | LSB interactivo en web sin instalar |

## Limitaciones

| Limitación | Detalle |
|---|---|
| Solo PNG/BMP | No JPEG (lossy), GIF animado, WAV, MP3 |
| Solo LSB espacial | No DCT/DWT/DFT (ver [[Estegoanálisis - Detección y ContraMedidas]] §2.7) |
| No descifra | Si el payload venía cifrado, devuelve bytes ilegibles |
| No hace estegoanálisis | No calcula χ²/RS: es extractor, no detector estadístico |
| Rendimiento | `-a` puede tardar en imágenes > 10 MP |

## Errores comunes

1. **Ejecutar zsteg sobre JPEG** y concluir "no hay nada" → convertir no sirve (el LSB original ya se perdió); atacar con steghide/stegdetect.
2. **Confiar en el primer `text:` con `-a`**: filtrar y validar siempre.
3. **Olvidar `metadata`**: muchísimos CTF esconden la flag en un chunk `tEXt` y ni siquiera usan LSB.
4. **Probar solo `b1`**: si el embebido usó 2–4 bits o canal alfa, hace falta `-a`.
5. **No probar semillas** cuando el texto sale cifrado: `rand:1337` y compañía son frecuentes.
6. **Dejar basura en disco**: el pipeline crea decenas de `.bin`; revisar con `file` antes de abrirlos todos.

## Cheatsheet

```bash
zsteg imagen.png                       # 90% de los casos
zsteg -a imagen.png                    # exhaustivo (si el rápido no rinde)
zsteg -E "metadata" imagen.png         # solo chunks PNG
zsteg -E "b1,rgb,lsb,xy" imagen.png > out.bin
for s in 0 1 123 1337 31337; do zsteg -E "b1,rgb,lsb,rand:$s" img.png | strings | grep flag; done
file out.bin && strings -n 6 out.bin | head -20
```

**Checklist CTF**: `file` → `zsteg` → `zsteg -a` → extraer `text:`/`file:` → `file`/`strings` → recursión (`base64 -d`, `unzip`, `binwalk`) → si binario cifrado → stegsolve visual / steghide.

## Casos de referencia

| Reto | Técnica | Comando ganador |
|---|---|---|
| PicoCTF "So Meta" | Metadata PNG | `zsteg -E "metadata" img.png` |
| HTB "Stego" | LSB b1 RGB | `zsteg -E "b1,rgb,lsb,xy" img.png` |
| CTFlearn "Hidden" | LSB b1 RGBA | `zsteg -E "b1,rgba,lsb,xy" img.png` |
| RingZer0 "Stegano" | 2 bits/canal | `zsteg -E "b1-2,rgb,lsb,xy" img.png` |
| Custom "Random" | Semilla 1337 | `zsteg -E "b1,rgb,lsb,rand:1337" img.png` |

## Referencias

- Repositorio: <https://github.com/zed-0xff/zsteg>
- RubyGems: <https://rubygems.org/gems/zsteg>
- PNG specification (chunks): <https://www.w3.org/TR/png-3/>
- StegOnline (LSB interactivo): <https://stegonline.georgeomura.com>

## Relacionadas

- [[Esteganografía - Conceptos, Herramientas y Práctica]] — teoría LSB y flujos completos
- [[Estegoanálisis - Detección y ContraMedidas]] — χ²/RS y el resto de detección
- [[Steghide - Uso Avanzado]] — cuando el formato es JPEG/BMP/WAV
- [[Stegseek - Guía Rápida]] — cracking de passphrases steghide
