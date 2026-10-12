---
aliases:
  - Estegoanálisis
  - Steganálisis
  - Detección de Esteganografía
  - Steganalysis
tags:
  - sec/stego
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Esteganografía - Conceptos, Herramientas y Práctica]]"
  - "[[Steghide - Uso Avanzado]]"
  - "[[Stegseek - Guía Rápida]]"
  - "[[Zsteg - Análisis LSB PNG]]"
  - "[[Wireshark y Tshark]]"
fuente: "https://www.wsj.net/public/resources/images/BN-BP351_stego_J_20141219144947.jpg"
---

# Estegoanálisis: Detección y Contramedidas

**Estegoanálisis** es la disciplina forense que detecta, estima y — cuando es posible — extrae información oculta en un portador. Su lema es *"no hay esteganografía perfecta, solo estegoanálisis insuficiente"*: toda técnica de embedding introduce un sesgo medible (visual, estadístico, estructural o de firma), y el trabajo del analista es encontrar ese sesgo.

> [!info] Objetivos del analista, en orden
> 1. **Detección**: ¿hay payload oculto? (presencia/ausencia)
> 2. **Estimación**: ¿cuántos bits se usaron y con qué algoritmo?
> 3. **Extracción**: ¿cuál es el payload? (con clave o rompiendo el esquema)
> 4. **Atribución**: ¿qué herramienta lo produjo? (firmas → reglas de detección)

## 1. Clasificación de ataques (según conocimiento del analista)

| Tipo | Qué tiene el analista | Poder | Aplicación real |
|---|---|---|---|
| **Solo estego-objeto** (*blind*) | Solo el archivo sospechoso | Detectar presencia, estimar capacidad | CTF, forense, DLP |
| **Conocida cover** | Cover original + estego | Análisis diferencial → extracción **exacta** sin clave | Imágenes con watermark/originales en la web |
| **Conocido payload** | Payload + estego | Ingeniería inversa del algoritmo y semilla | Campañas conocidas (firmas APT) |
| **Cover/payload elegidos** | Control total (lab) | Caracterizar la herramienta y crear firmas | Investigación, entrenamiento de detectores |

> [!tip] La asimetría clave
> Un atacante que **reutiliza** una cover publicada (foto de perfil, banner de web, logo de GitHub) convierte un ataque *blind* en un ataque de *cover conocida*: `diff` byte a byte revela exactamente qué cambió. Nunca reutilices covers; nunca des por limpia una cover que exista en otro sitio.

## 2. Detección estadística (ciega)

### 2.1 El principio: el embedding destruye distribuciones naturales

Las imágenes naturales tienen histogramas **suaves**: los valores vecinos (par/impar, `2k`/`2k+1`) se distribuyen según una curva continua. El LSB de sustitución fuerza la mitad de los píxeles a valores arbitrarios y **aplanamiento** esas parejas. Todos los tests clásicos miden esa deformación.

### 2.2 Chi-cuadrado (χ²) — Westfeld

**Mecánica**: se agrupan los valores del histograma en parejas `[2k, 2k+1]`. Si la imagen está limpia, la proporción dentro de cada pareja sigue la forma de la curva; si hay LSB embedding, ~50% de los píxeles de la pareja fue forzado y **ambos valores convergen a la media**. El test χ² mide cuánto se desvía la distribución observada de la esperada:

```
χ² = Σ (O_i − E_i)² / E_i     con p < 0,05 → rechaza "imagen limpia"
```

| Señal | Interpretación |
|---|---|
| `p` muy bajo (χ² alto) en los primeros cientos de pares | Imagen limpia → el embedding no ha tocado esa zona |
| `p` **alto** (χ² bajo) con histograma aplanado en parejas | **Sospechoso**: consistente con LSB embedding |
| χ² alto en TODAS las parejas | Imagen realmente natural o embedding con **LSB matching** (±1) |

```python
# χ² por canal — rápido y suficiente como triaje
from PIL import Image
import numpy as np
from scipy.stats import chisquare

img = np.array(Image.open("sospechosa.png"))
lsb = img & 1
for c, name in enumerate(["R", "G", "B"]):
    hist = np.bincount(lsb[:, :, c].flatten(), minlength=2)
    chi2, p = chisquare(hist)
    verdict = "SOSPECHOSO" if p < 0.05 else "plausible limpio"
    print(f"{name}: χ²={chi2:.2f}, p={p:.6f} → {verdict}")
```

> [!warning] Falso negativo típico del χ²
> El test asume **sustitución** de LSB. Con **LSB matching** (±1 aleatorio) o embedding adaptativo la correlación par/impar se mantiene y χ² **no acusa**: una imagen "limpia" según χ² no está exenta. Para eso existen RS/SPA.

### 2.3 RS Analysis (Fridrich)

**Mecánica**: se particiona la imagen en grupos de *m* píxeles consecutivos y se define una función de **correlación suave** `f` (cuánto "ruido" tiene el grupo). Se aplican permutaciones de flujo:

| Operación | Significado |
|---|---|
| **M (mask)** | Invertir la LSB de todos los píxeles del grupo (simula embedding) |
| **M⁻¹ (inverse mask)** | Deshacer la inversión (idempotente para M) |
| **S (shift)** | Mover el valor a su vecino par (pares → pares, impares → impares) |

Se cuentan cuatro cantidades: `R_M`, `S_M`, `R_SM`, `S_SM` (grupos *regulares* / *singulares* con y sin máscara). En una imagen limpia se cumple `R_M ≈ R_SM` y `S_M ≈ S_SM`; con payload embebido, **`R_M − S_M` se separa de `R_SM − S_SM`** de forma proporcional a la cantidad de bits. Ventaja sobre χ²: funciona con LSB matching y estima la **fracción de píxeles modificados**.

```bash
# Implementaciones de referencia
# - stegdetect (C, clásico)     # - RS / SPA en herramientas académicas (StegExpose agrega varios)
stegdetect -t sospechosa.jpg   # calibración por domain (jsteg/outguess/f5/steghide)
```

### 2.4 Sample Pair Analysis (SPA)

Variante de RS: analiza la **correlación entre pares de píxeles adyacentes** (par→par, par→impar, impar→par, impar→impar) y resuelve el sistema de ecuaciones que liga esas tasas con la tasa de embedding. Más sensible que χ² en imágenes con poco embedding; coste computacional mayor.

### 2.5 Entropía (global y por bloques)

```bash
ent sospechoso.png
# Entropy = 7.9999 bits/byte → casi aleatorio: cifrado, comprimido o estego
# Entropy ≈ 7.4–7.8        → imagen natural típica
# Entropy < 7.0            → textura simple, poco probable que lleve payload grande
```

```python
# Entropía por bloques → localiza la REGIÓN modificada
from PIL import Image
import numpy as np
img = np.array(Image.open("sospechosa.png"))
block = 64
for y in range(0, img.shape[0], block):
    for x in range(0, img.shape[1], block):
        patch = img[y:y+block, x:x+block].flatten()
        _, counts = np.unique(patch, return_counts=True)
        p = counts / len(patch)
        ent = -np.sum(p * np.log2(p + 1e-10))
        if ent > 7.95:
            print(f"Bloque ({x},{y}): entropía={ent:.4f} SOSPECHOSO")
```

### 2.6 Resumen de tests estadísticos

| Test | Qué detecta | Herramienta | Falso negativo típico |
|---|---|---|---|
| **χ²** | Sustitución LSB (parejas aplanadas) | custom, stegdetect | LSB matching, embedding adaptativo |
| **RS** | Sustitución + matching (±1) | académico, StegExpose | Embedding de baja tasa en textura |
| **SPA** | Correlación par→par | académico | Imágenes muy pequeñas |
| **Histograma visual** | Picos par/impar | stegsolve, GIMP | Cualquier método de baja tasa |
| **Entropía** | Zonas anómalas (cifrado/estego) | `ent`, scripts | Comprimido natural (JPEG) |

### 2.7 Análisis en el dominio JPEG (DCT)

| Técnica | Mecánica | Detecta |
|---|---|---|
| **Calibración** | Comparar histogramas DCT del archivo contra una copia recomprimida "limpia" generada por la propia herramienta | JSteg, OutGuess, F5, nsF5 |
| **Histograma de coeficientes AC** | Saltos en `0 → ±1 → ±2` (JSteg) o histograma "conservado" (OutGuess) | Cada familia de algoritmo |
| **Histograma duplicado** | Duplicación de picos 0/±1 (duplicación de frecuencias) | F5 y derivados |

```bash
stegdetect -v sospechosa.jpg
# sospechosa.jpg : jsteG(??)     ← JSteg / OutGuess probable
# sospechosa.jpg : f5(??)        ← F5 / nsF5 probable
# sospechosa.jpg : steghide(??)  ← steghide en BMP/JPEG
# sospechosa.jpg : not found     ← sin firma: queda el análisis estadístico
```

## 3. Detección por firmas (conocimiento del algoritmo)

| Herramienta | Firma / artefacto | Detector |
|---|---|---|
| **steghide** | Cabecera y estructura propietaria (`steghide` magic) + estándar de datos embebidos | `steghide info`, `stegdetect`, `xxd \| grep -i steghide` |
| **openstego** | Chunks/marcas PNG propias | `pngcheck`, `zsteg` |
| **JSteg** | LSB secuencial sobre coeficientes AC sin cifrado | `stegdetect` |
| **OutGuess** | Preserva estadísticas DCT → huecos en la calibración | `stegdetect` |
| **F5 / nsF5** | Matrix embedding + permutación | `stegdetect` |
| **stegcloak** | Zero-width (ZWSP, ZWNJ, ZWJ) en texto | `xxd`, scripts Unicode |
| **Appending** | Datos tras `FFD9` / `IEND` | `binwalk`, `xxd` |

```bash
# stegspy — búsqueda de firmas por offset (compilar desde fuente)
git clone https://github.com/b3dk7/stegspy.git && cd stegspy && make
stegspy sospechoso.jpg
# [+] steghide signature found at offset 0x12345
# [+] Estimated payload size: 1024 bytes
```

## 4. Ataque visual (manual)

### 4.1 Bit-planes con stegsolve / stegonline

```bash
java -jar stegsolve.jar
# File → Open → sospechosa.png
# Analyze → Bit Planes → R0, G0, B0, A0 … (0 = LSB)
# también: Red/Green/Blue/Alpha planes, Data Extract, Plane Slice, XOR
```

| Observación en el bit-plane 0 | Interpretación |
|---|---|
| Ruido aleatorio uniforme | Cover limpia **o** embedding con clave/aleatorización |
| Patrones (texto, líneas, rejilla, firma) | LSB secuencial sin PRNG → extracción trivial |
| Solo canal Azul modificado | Embedding selectivo (muchas herramientas solo usan B) |
| Ruido también en bits 1–3 | Multi-bit embedding (capacidad alta) |
| Alpha 0 con estructura | Payload en el canal alfa (PNG RGBA) |

### 4.2 Diferencial con la cover original

```bash
convert cover_original.png cover_stego.png -compose difference -composite diff.png
convert diff.png -auto-level -brightness-contrast 50x100 diff_enhanced.png
# diff.png claro ⇒ píxeles alterados; magnitud = bits tocados
```

```bash
# Byte a byte (sirve también con formatos "iguales")
cmp -l cover.jpg stego.jpg | head        # offsets con bytes distintos
```

## 5. Análisis estructural / de formato

```bash
# PNG: orden y CRC de chunks; chunk desconocido o IDAT extra = bandera roja
pngcheck -v imagen.png
# IHDR/PLTE/IDAT… IEND estándar; tEXt/zTXt/iTXt o chunks 'xxxx' privados → extraer

# JPEG: segmentos APPn/COM anómalos y datos tras EOI (FFD9)
jpegdump.py imagen.jpg                    # Didier Stevens
exiftool imagen.jpg                       # Comment largo, thumbnail anómalo, MakerNotes
exiftool -b -Comment imagen.jpg > extraido.bin

# Binwalk: firmas embebidas + entropía por offset
binwalk -e imagen.jpg                     # → _imagen.jpg.extracted/
binwalk -E imagen.jpg                     # gráfico: picos = zonas cifradas/opacas
```

## 6. Contramedidas: ambas direcciones

### 6.1 Para quien oculta (red team / OPSEC)

| Técnica | Efectividad | Coste | Nota |
|---|---|---|---|
| Cifrar antes (AES-256 + PBKDF2) | ★★★★★ | Bajo | Obligatoria |
| Cover única, alta entropía natural | ★★★★★ | Bajo | Elimina el ataque de cover conocida |
| Usar ≤ 10–20% de la capacidad | ★★★★☆ | Bajo | Menos sesgo estadístico |
| LSB matching (±1) en vez de sustitución | ★★★★☆ | Medio | Rompe χ² |
| Embedding adaptativo (bordes/textura, PVD) | ★★★★★ | Alto | Rompe χ² y RS en baja tasa |
| DCT/DWT + matrix embedding + permutación | ★★★★★ | Muy alto | Robusto y con firma mínima |
| Esteganografía adversarial (deep learning) | ★★★★★ | Muy alto | Investigación (ALASKA2) |
| Limpieza de metadatos | ★★★★☆ | Bajo | `exiftool -all= -overwrite_original` |

**LSB matching — por qué evade el χ²:**

```python
# Sustitución (detectable): 50% de los píxeles cambia de paridad forzada
#    LSB != bit → r = (r & ~1) | bit
# Matching (difícil): solo toca los píxeles cuya paridad NO coincide, ±1 aleatorio
#    LSB != bit → r = r + 1  o  r - 1  (25% de píxeles afectados, paridad conservada
#    en promedio ⇒ el histograma de parejas NO se aplana)
import random
def lsb_match(v, bit):
    if (v & 1) == bit:
        return v
    step = random.choice((1, -1))
    return max(0, min(255, v + step))
```

**Embedding adaptativo (sketch):**

```python
import cv2, numpy as np
gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
edges = cv2.Canny(gray, 30, 60)
mask = cv2.dilate(edges, np.ones((3, 3), np.uint8), iterations=1) > 0
# embed SOLO donde mask == True (bordes, textura) — el fondo plano queda intacto
```

### 6.2 Para quien detecta (blue team / defensa)

| Acción | Efecto |
|---|---|
| DLP con módulo estego (stegdetect/zsteg en el gateway) | Bloquea archivos con firmas conocidas |
| Re-encode obligatorio de imágenes subidas (thumbnails, recompresión) | **Destruye LSB y appending** de forma barata |
| Prohibir formatos con pérdidas perdidas (permitir solo JPEG/WebP re-encoded) | Neutraliza la esteganografía espacial |
| Alerta por tamaño/anomalía de binarios en proxy | Payloads grandes |
| Threat intel de imágenes hash-idénticas a campañas conocidas | Atribución (APT28, Stegoloader) |
| Análisis forense con el pipeline de §7 | Respuesta a incidentes |

> [!tip] La defensa más barata contra la esteganografía
> **Re-codificar todo lo que entra**: un pipeline que recorte, recompile a JPEG calidad 80 y regenere los EXIF elimina LSB, chunks de texto, appending y polyglots sin ningún detector estadístico. El coste es perder el archivo original (por eso no sirve para forense: ahí se preserva el original y se analiza).

## 7. Pipeline forense completo (script)

```bash
#!/bin/bash
# forensics_stego.sh — pipeline de detección en 7 pasos
TARGET="$1"; OUTDIR="forensics_$(basename "$TARGET")"; mkdir -p "$OUTDIR"

echo "[1] Identificación";  file "$TARGET" > "$OUTDIR/01_file.txt";  exiftool "$TARGET" > "$OUTDIR/02_exif.txt"
echo "[2] Strings";         strings -n 6 "$TARGET" > "$OUTDIR/03_strings.txt"
grep -iE "flag|pass|secret|key|hidden" "$OUTDIR/03_strings.txt" > "$OUTDIR/03b.txt"
echo "[3] Estructura"
case "$TARGET" in
  *.png)          pngcheck -v "$TARGET" > "$OUTDIR/04_pngcheck.txt" ;;
  *.jpg|*.jpeg)   jpegdump.py "$TARGET" > "$OUTDIR/04_jpeg.txt" 2>/dev/null ;;
esac
echo "[4] Binwalk";  binwalk -e "$TARGET" --directory "$OUTDIR/05_binwalk" 2>/dev/null
echo "[5] Estego específico"
if [[ "$TARGET" =~ \.(png|bmp)$ ]]; then
  zsteg -a "$TARGET" > "$OUTDIR/06_zsteg.txt" 2>/dev/null
  zsteg "$TARGET" 2>/dev/null | grep "text:" | awk '{print $1}' | while read -r m; do
    zsteg -E "$m" "$TARGET" > "$OUTDIR/06_${m//,/_}.bin" 2>/dev/null
  done
fi
if [[ "$TARGET" =~ \.(jpg|jpeg|bmp|wav|au)$ ]]; then
  steghide info "$TARGET" 2>&1 | head -10 > "$OUTDIR/06_steghide.txt"
  stegdetect "$TARGET" > "$OUTDIR/06_stegdetect.txt" 2>/dev/null
  stegspy "$TARGET" > "$OUTDIR/06_stegspy.txt" 2>/dev/null
fi
echo "[6] Entropía"; ent "$TARGET" > "$OUTDIR/07_ent.txt" 2>/dev/null
echo "[7] Visual (manual): stegsolve / stegonline bit-planes"
echo "[+] Resultados en $OUTDIR/"; ls -la "$OUTDIR/"
```

## 8. Metodología CTF: de la sospecha a la flag

| Paso | Acción | Comando clave |
|---|---|---|
| 1 | Triage rápido de formato | `file`, `exiftool`, `strings -n 6` |
| 2 | Contenedores ocultos | `binwalk -e`, `pngcheck -v`, revisar tras EOI |
| 3 | LSB automatizado | `zsteg -a` → extraer cada `text:`/`file:` candidato |
| 4 | steghide | `steghide info` → `stegseek --crack ... rockyou.txt` |
| 5 | Visual | stegsolve bit-planes, XOR, Data Extract |
| 6 | Estadístico (si hay duda) | χ², RS, `ent` — confirma o descarta |
| 7 | Anidar | El extraído es otro archivo → vuelve al paso 1 (`file`/`binwalk`) |
| 8 | Documentar | Herramienta + flag + hash SHA-256 del original y del extraído |

Ver flujo completo de herramientas: [[Esteganografía - Conceptos, Herramientas y Práctica]], [[Zsteg - Análisis LSB PNG]], [[Stegseek - Guía Rápida]].

## 9. Casos reales

| Caso | Vector | Técnica | Cómo se detectó |
|---|---|---|---|
| **APT28 / Fancy Bear** | Imágenes legítimas subidas a Twitter, GitHub, Imgur | LSB en PNG + AES | Anomalías estadísticas + correlación con dominios C2 de threat intel |
| **Stegoloader (2017)** | PNG en repo público de GitHub | LSB RGB + XOR key | zsteg + fuerza bruta de la XOR key |
| **Exfiltración interna (insider)** | Fotos "team building" a Drive/SharePoint | steghide en JPEG + passphrase derivada de fecha | DLP + `steghide info` + análisis conductual |
| **Malware genérico** | Canales DNS encubiertos | Subdominios con datos | Entropía y longitud de consultas DNS (ver §3.5 de la nota de conceptos) |

## 10. Machine learning (estado del arte)

| Modelo | Precisión típica | Dataset | Uso |
|---|---|---|---|
| **Xu-Net** | ~93% | BOSSBase | Baseline académico |
| **Ye-Net** | ~94% | BOSSBase | Research |
| **SRNet** (Spatial Rich Model + CNN) | ~95%+ en JPEG | BOSSBase/ALASKA2 | Competición BOSS/ALASKA |
| **SRNet/J-UNIWARD (selection-channel-aware)** | SOTA | ALASKA2 | Evade embedding adaptativo |

```python
# Esquema conceptual — requiere modelo entrenado + GPU
import torch
model = SRNet(); model.load_state_dict(torch.load("srnet_jpeg.pth")); model.eval()
# entrada: coeficientes DCT del JPEG (no píxeles)
# salida: probabilidad estego > 0.5 → detectado
```

> [!warning] Los detectores ML no son "out-of-the-box" para forense
> Están sobreajustados al dataset y al algoritmo con los que se entrenaron; con una herramienta no vista (o LSB matching) su precisión cae. Úsalos como **voto adicional**, nunca como veredicto único, y conserva siempre el original con su hash (cadena de custodia).

## 11. Errores comunes del analista

1. **Dar por limpio un archivo con `stegdetect` negativo**: solo cubre firmas JPEG conocidas; LSB en PNG ni lo mira.
2. **Olvidar el paso de recursión**: el payload suele ser `zip` → `png` → `flag`. `file` tras cada extracción.
3. **Confiar en un solo test estadístico**: χ² tiene falsos negativos con LSB matching; RS tiene poca potencia con pocas muestras.
4. **Recomprimir antes de analizar**: destruye la evidencia. Trabajar siempre sobre una copia con hash verificado.
5. **Ignorar los metadatos**: `Comment`, `Software`, `Thumbnail` son el vector más frecuente en CTF y el más barato de revisar.
6. **No registrar parámetros**: sin anotar el método exacto (`zsteg -E "b1,rgb,lsb,xy"`), el informe no es reproducible.

## 12. Cheatsheet

```bash
# Triaje
file IMG; exiftool IMG; strings -n 6 IMG | grep -iE 'flag|pass|key'
binwalk -e IMG; pngcheck -v IMG.png; ent IMG

# Estadístico / firma
stegdetect -v IMG.jpg; stegspy IMG.jpg

# Visual
java -jar stegsolve.jar        # bit-planes R0/G0/B0/A0, XOR, Data Extract

# Diferencial (cover conocida)
cmp -l cover.jpg stego.jpg | head
convert cover.png stego.png -compose difference -composite diff.png

# Contramedida defensiva (ingesta)
mogrify -quality 85 -strip *.jpg     # re-codificar + limpiar EXIF
```

## 13. Referencias

- Fridrich, J. et al. — *Steganalysis of LSB Encoding in Color Images* (2001)
- Fridrich, J. — *Feature-Based Steganalysis for JPEG Images* (2002)
- Ker, A. — *Steganalysis of LSB Matching in Grayscale Images* (2005)
- Pevný, T. et al. — *Steganalysis by Subtractive Pixel Adjacency Matrix* (SPAM, 2010)
- Denemark, T. et al. — *Selection-Channel-Aware Steganography* (2014)
- Datasets: **BOSSBase** (10.000 covers) · **ALASKA2** (Kaggle, 2019)
- StegExpose (detectores varios): <https://github.com/b3dk7/StegExpose>
- Stegdetect (Niels Provos): <http://stegdetect.sourceforge.net/>
- StegSpy: <https://github.com/b3dk7/stegspy>

## Glosario

| Término | Significado |
|---|---|
| **Cover / Stego-objeto** | Portador limpio / portador con payload |
| **Embedding / Extraction** | Ocultar / recuperar |
| **RS Analysis** | Tests de grupos regulares/singulares (Fridrich) |
| **SPA** | Sample Pair Analysis |
| **Matrix embedding** | Codificación eficiente por síndrome (código de Hamming) |
| **LSB matching (±1)** | Embedding por ajuste ±1 en vez de sustitución |
| **DCT** | Discrete Cosine Transform (base de JPEG) |
| **Calibración** | Recomprimir y comparar histogramas para firmas JPEG |

## Relacionadas

- [[Esteganografía - Conceptos, Herramientas y Práctica]] — taxonomía y flujos de trabajo
- [[Steghide - Uso Avanzado]] — detección y extracción del formato steghide
- [[Stegseek - Guía Rápida]] — romper passphrases cuando hay datos steghide
- [[Zsteg - Análisis LSB PNG]] — extracción LSB en PNG/BMP
- [[Wireshark y Tshark]] — detección de canales encubiertos en red
