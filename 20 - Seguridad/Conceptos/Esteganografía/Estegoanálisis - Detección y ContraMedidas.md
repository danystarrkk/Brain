# Estegoanálisis: Detección y Contramedidas

> **Definición**: Ciencia de detectar, analizar y romper esteganografía. Objetivo: determinar si un objeto contiene información oculta, estimar capacidad, y si posible, extraerla sin clave.
>
> **Lema**: *"No hay esteganografía perfecta, solo estegoanálisis insuficiente"*

---

## 1. Clasificación de Ataques

| Tipo | Conocimiento | Objetivo |
|------|--------------|----------|
| **Solo Estego-objeto** | Solo archivo sospechoso | Detectar presencia, estimar payload |
| **Conocido Cover** | Cover original + stego | Análisis diferencial, extracción exacta |
| **Conocido Payload** | Payload conocido + stego | Reverse engineering del algoritmo |
| **Elegido Cover/Payload** | Control total (lab) | Caracterizar herramienta, crear firmas |

---

## 2. Detección Estadística (Ciegas - Solo Stego)

### 2.1 Análisis LSB (Imágenes)

| Test | Qué Detecta | Herramienta |
|------|-------------|-------------|
| **Chi-Cuadrado (χ²)** | Distribución uniforme LSB (pares vs impares) | `stegdetect`, custom Python |
| **RS Analysis** | Grupos de píxeles regulares/singulares | `stegdetect`, `stegspy` |
| **Sample Pair Analysis (SPA)** | Correlación entre pares adyacentes | Academic tools |
| **Histogramas LSB** | Picos en valores pares/impares | Visual (stegsolve) |

```python
# Chi-square test rápido (Python)
from PIL import Image
import numpy as np
import scipy.stats as stats

img = np.array(Image.open('sospechosa.png'))
lsb = img & 1
for c, name in enumerate(['R','G','B']):
    hist = np.bincount(lsb[:,:,c].flatten(), minlength=2)
    chi2, p = stats.chisquare(hist)
    print(f'{name}: χ²={chi2:.2f}, p={p:.6f} {"SOSPECHOSO" if p<0.05 else "OK"}')
# p < 0.05 → distribución no uniforme → probable LSB embedding
```

### 2.2 Análisis JPEG (Dominio DCT)

| Test | Herramienta | Qué Detecta |
|------|-------------|-------------|
| **Calibración** | `stegdetect` | JSteg, OutGuess, F5, nsF5 |
| **Distribución coeficientes DCT** | `stegdetect` | Embedding en coeficientes AC |
| **Histogramas DCT** | Custom | Anomalías en coeficientes bajos |

```bash
# stegdetect - detección automática JPEG
stegdetect desafio.jpg
# desafio.jpg : jsteG(??)  <- JSteg/OutGuess probable
# desafio.jpg : f5(??)     <- F5/nsF5 probable
# desafio.jpg : steghide(??) <- steghide

# Modo verbose
stegdetect -v desafio.jpg
# Muestra probabilidades por algoritmo
```

### 2.3 Análisis de Entropía

```bash
# ent - entropía de Shannon (bits/byte)
ent sospechoso.png
# Entropy = 7.9999 bits/byte → muy alta (cifrado/comprimido/estego)
# Entropy ~ 7.5-7.8 → normal imagen natural
# Entropy < 7.0 → baja complejidad, poco probable estego

# Por bloques (detectar regiones modificadas)
python3 -c "
from PIL import Image
import numpy as np
img = np.array(Image.open('sospechosa.png'))
h, w = img.shape[:2]
block = 64
for y in range(0, h, block):
    for x in range(0, w, block):
        patch = img[y:y+block, x:x+block].flatten()
        _, counts = np.unique(patch, return_counts=True)
        probs = counts / len(patch)
        ent = -np.sum(probs * np.log2(probs + 1e-10))
        if ent > 7.95:
            print(f'Bloque ({x},{y}): entropía={ent:.4f} SOSPECHOSO')
"
```

---

## 3. Detección por Firmas (Conocidas)

### 3.1 Herramientas con Firmas Conocidas

| Herramienta | Firma / Artefacto | Detector |
|-------------|-------------------|----------|
| **steghide** | Header "steghide" + estructura propietaria | `stegdetect`, `stegspy`, `steghide info` |
| **openstego** | Chunk PNG "oStego" / marca en header | `zsteg`, `pngcheck` |
| **JSteg** | LSB secuencial sin cifrado | `stegdetect` |
| **OutGuess** | Preserva estadísticas DCT | `stegdetect` |
| **F5 / nsF5** | Matrix embedding, permutación | `stegdetect` |
| **stegcloak** | Unicode zero-width chars (ZWSP, ZWNJ) | `strings`, `xxd`, custom |

### 3.2 stegspy - Escáner de Firmas

```bash
# Instalación (C, requiere compilación)
git clone https://github.com/b3dk7/stegspy.git
cd stegspy && make
sudo cp stegspy /usr/local/bin/

# Uso
stegspy sospechoso.jpg
# [+] steghide signature found at offset 0x12345
# [+] Estimated payload size: 1024 bytes
```

---

## 4. Análisis Visual (Manual)

### 4.1 Bit-Planes (stegsolve / stegonline)

```bash
# stegsolve.jar (Java)
java -jar stegsolve.jar
# File → Open → sospechosa.png
# Analyze → Bit Planes → Red 0, Green 0, Blue 0, Alpha 0
# Buscar: patrones, texto, ruido estructurado en LSB (bit 0)
```

**Interpretación**:
- **Ruido aleatorio uniforme** → Normal (sin estego o estego bien hecho)
- **Patrones visibles** (líneas, texto, formas) → LSB embedding simple
- **Solo algunos canales** → Embedding selectivo (ej. solo Blue)
- **Bit 1-3 también** → Multi-bit embedding

### 4.2 Diferencial (Cover Original vs Stego)

```bash
# ImageMagick - diferencia visual
convert cover_original.png cover_stego.png -compose difference -composite diff.png
# Brillo en diff.png = píxeles modificados

# Amplificar diferencia
convert diff.png -auto-level -brightness-contrast 50x100 diff_enhanced.png
```

---

## 5. Análisis Estructural / Formato

### 5.1 PNG - Chunks Anómalos

```bash
# pngcheck - análisis completo chunks
pngcheck -v imagen.png
# Buscar:
# - Chunks desconocidos (cUSTOM, tEXt extra, etc.)
# - Chunks fuera de orden (IHDR, PLTE, IDAT, IEND estándar)
# - Múltiples IDAT (normal) vs IDAT + datos extra al final
# - CRC errors (indica modificación)

# Extraer chunks sospechosos
pngcheck -7 -c imagen.png  # Solo chunks textuales
```

### 5.2 JPEG - Segmentos SOS/APPn

```bash
# JPEG structure
jpegdump.py imagen.jpg  # (de Didier Stevens)
# Buscar: APPn extra, COM largo, datos tras EOI (0xFFD9)

# Exiftool
exiftool imagen.jpg
# - Comment muy largo
# - Thumbnail anómalo
# - Datos en MakerNotes
```

### 5.3 Binwalk - Archivos Embebidos

```bash
# Firmas de archivos conocidos
binwalk -e imagen.jpg
# _imagen.jpg.extracted/  → archivos recuperados

# Entropía por offset
binwalk -E imagen.jpg
# Gráfico entropía → picos = datos ocultos/cifrados
```

---

## 6. Contramedidas (Anti-Estegoanálisis)

### 6.1 Para el Que Oculta (Red Team / OPSEC)

| Técnica | Efectividad | Costo |
|---------|-------------|-------|
| **Cifrar ANTES (AES-256)** | ★★★★★ | Bajo |
| **Cover con alta entropía natural** | ★★★★☆ | Medio |
| **LSB matching (±1) en vez de reemplazo** | ★★★★☆ | Medio |
| **Embedding adaptativo (PVD, edge areas)** | ★★★★★ | Alto |
| **Dominio frecuencia (DCT/DWT) + matrix embedding** | ★★★★★ | Muy Alto |
| **Steganografía adversarial (Deep Learning)** | ★★★★★ | Muy Alto |
| **Cover única por operación** | ★★★★★ | Bajo |
| **Limpieza metadata EXIF** | ★★★★☆ | Bajo |
| **Tamaño payload << capacidad máxima** | ★★★★☆ | Bajo |

### 6.2 Implementación: LSB Matching (±1)

```python
# En lugar de forzar LSB = bit_mensaje (cambia 50% píxeles)
# LSB matching: si LSB != bit → ±1 aleatorio (cambia 25% píxeles)
# Más difícil detección chi-square

from PIL import Image
import random

def lsb_match_embed(cover_path, payload_bits, output_path):
    img = Image.open(cover_path)
    pixels = img.load()
    w, h = img.size
    idx = 0
    
    for y in range(h):
        for x in range(w):
            if idx >= len(payload_bits): break
            r, g, b = pixels[x, y]
            target = int(payload_bits[idx])
            
            # Canal R
            if (r & 1) != target:
                r = r + 1 if random.random() < 0.5 else r - 1
                r = max(0, min(255, r))
            idx += 1
            if idx >= len(payload_bits): break
            
            # Canal G
            if (g & 1) != target:
                g = g + 1 if random.random() < 0.5 else g - 1
                g = max(0, min(255, g))
            idx += 1
            if idx >= len(payload_bits): break
            
            # Canal B
            if (b & 1) != target:
                b = b + 1 if random.random() < 0.5 else b - 1
                b = max(0, min(255, b))
            idx += 1
            
            pixels[x, y] = (r, g, b)
        if idx >= len(payload_bits): break
    
    img.save(output_path, 'PNG')
```

### 6.3 Embedding Adaptativo (Solo Bordes/Textura)

```python
# Solo modificar píxeles en zonas de alta varianza (bordes, textura)
# Herramientas: openstego (plugin adaptive), custom

import cv2
import numpy as np

def get_edge_mask(img, threshold=30):
    gray = cv2.cvtColor(img, cv2.COLOR_RGB2GRAY)
    edges = cv2.Canny(gray, threshold, threshold*2)
    # Dilatar para zona alrededor de bordes
    kernel = np.ones((3,3), np.uint8)
    return cv2.dilate(edges, kernel, iterations=1) > 0

# Solo embed donde mask == True
```

---

## 7. Pipeline de Análisis Forense (Blue Team)

```bash
#!/bin/bash
# forensics_stego.sh - Pipeline completo

TARGET="$1"
OUTDIR="forensics_$(basename "$TARGET")"
mkdir -p "$OUTDIR"

echo "[1] Identificación básica"
file "$TARGET" > "$OUTDIR/01_file.txt"
exiftool "$TARGET" > "$OUTDIR/02_exiftool.txt"

echo "[2] Strings y firmas"
strings "$TARGET" > "$OUTDIR/03_strings.txt"
grep -i "flag\|pass\|secret\|key\|hidden" "$OUTDIR/03_strings.txt" > "$OUTDIR/03b_interesting.txt"

echo "[3] Estructura formato"
case "$TARGET" in
    *.png) pngcheck -v "$TARGET" > "$OUTDIR/04_pngcheck.txt" ;;
    *.jpg|*.jpeg) jpegdump.py "$TARGET" > "$OUTDIR/04_jpegdump.txt" 2>/dev/null || true ;;
esac

echo "[4] Binwalk - archivos embebidos"
binwalk -e "$TARGET" --directory "$OUTDIR/05_binwalk" 2>/dev/null

echo "[5] Esteganografía específica"
# PNG/BMP → zsteg
if [[ "$TARGET" =~ \.(png|bmp)$ ]]; then
    zsteg -a "$TARGET" > "$OUTDIR/06_zsteg.txt" 2>/dev/null
    # Extraer candidatos
    zsteg "$TARGET" 2>/dev/null | grep "text:" | awk '{print $1}' | while read m; do
        zsteg -E "$m" "$TARGET" > "$OUTDIR/06_${m//,/_}.bin" 2>/dev/null
    done
fi

# JPEG → steghide, stegdetect
if [[ "$TARGET" =~ \.(jpg|jpeg)$ ]]; then
    steghide info "$TARGET" 2>&1 | head -10 > "$OUTDIR/06_steghide_info.txt"
    stegdetect "$TARGET" > "$OUTDIR/06_stegdetect.txt"
    stegspy "$TARGET" > "$OUTDIR/06_stegspy.txt" 2>/dev/null || true
fi

# WAV/AU → steghide
if [[ "$TARGET" =~ \.(wav|au)$ ]]; then
    steghide info "$TARGET" 2>&1 > "$OUTDIR/06_steghide_audio.txt"
fi

echo "[6] Análisis entropía"
ent "$TARGET" > "$OUTDIR/07_ent.txt"

echo "[7] Visual (requiere GUI)"
# stegsolve, stegonline manual

echo "[+] Resultados en: $OUTDIR/"
ls -la "$OUTDIR/"
```

---

## 8. Machine Learning / Deep Learning (State of the Art)

| Enfoque | Precisión | Uso |
|---------|-----------|-----|
| **SRNet** (Spatial Rich Net) | ~95%+ en JPEG | Research, competiciones |
| **Xu-Net** | ~93% | Baseline académico |
| **Ye-Net** | ~94% | Research |
| **Custom CNN** | Variable | Entrenado en dataset específico |

```python
# Ejemplo conceptual (requiere modelo entrenado)
import torch
from srnet import SRNet

model = SRNet()
model.load_state_dict(torch.load('srnet_jpeg.pth'))
model.eval()

# Preprocesar JPEG (coeficientes DCT)
# output = model(dct_coefficients)
# output > 0.5 → stego detectado
```

> **Nota**: Requiere datasets de entrenamiento (BOSSBase, ALASKA2) y GPU. No listo "out-of-the-box" para forense operativo.

---

## 9. Casos de Estudio Reales

### Caso A: APT28 (Fancy Bear) - Steganografía en Imágenes Legítimas
- **Vector**: Imágenes subidas a sitios legítimos (Twitter, GitHub, Imgur)
- **Técnica**: LSB en PNG + cifrado AES-256
- **Detección**: Anomalías estadísticas + threat intel (dominios C2)

### Caso B: Malware "Stegoloader" (2017)
- **Vector**: PNG en repositorio GitHub público
- **Técnica**: Payload ejecutable en LSB RGB + XOR key
- **Extracción**: zsteg + XOR bruteforce

### Caso C: Exfiltración Corporativa (Insider)
- **Vector**: Fotos de "team building" subidas a Drive/SharePoint
- **Técnica**: steghide en JPEG + passphrase derivada de fecha evento
- **Detección**: DLP + stegdetect + análisis conductual

---

## 10. Checklist Detección Operativa

```
[ ] 1. file / exiftool / strings - reconocimiento básico
[ ] 2. binwalk -e - archivos embebidos / firmas
[ ] 3. Según formato:
     [ ] PNG/BMP → zsteg -a (LSB, metadata)
     [ ] JPEG → stegdetect + steghide info + stegspy
     [ ] WAV/AU → steghide info
     [ ] Otros → stegonline.com / stegsolve manual
[ ] 4. Análisis diferencial (si cover original disponible)
[ ] 5. Entropía global y por bloques (ent, script Python)
[ ] 6. Visual bit-planes (stegsolve/stegonline)
[ ] 7. Si hallazgo → extraer → file / strings / binwalk recursivo
[ ] 8. Si cifrado → identificar algoritmo → ataque cripto/fuerza bruta
[ ] 9. Documentar: herramienta, parámetros, hallazgos, hash archivos
[ ] 10. Cadena de custodia (hashes SHA256 original + extraídos)
```

---

## 11. Referencias Académicas y Técnicas

### Papers Fundamentales
- Fridrich, J. et al. *"Steganalysis of LSB Encoding in Color Images"* (2001)
- Fridrich, J. *"Feature-Based Steganalysis for JPEG Images"* (2002)
- Ker, A. *"Steganalysis of LSB Matching in Grayscale Images"* (2005)
- Pevný, T. et al. *"Steganalysis by Subtractive Pixel Adjacency Matrix"* (2010) - **SPAM features**
- Denemark, T. et al. *"Selection-Channel-Aware Steganography"* (2014) - **Content-adaptive**

### Datasets Benchmark
- **BOSSBase** (10,000 covers PGN) - Estándar académico
- **ALASKA2** (Kaggle, 2019) - 40k imágenes, 4 algoritmos
- **Break Our Steganographic System (BOSS)** - Challenge histórico

### Herramientas de Referencia
- **StegExpose** (Python, múltiples detectores) - https://github.com/b3dk7/StegExpose
- **Stegdetect** (Niels Provos) - Clásico JPEG
- **StegSpy** (b3dk7) - Firmas conocidas
- **SRNet / Xu-Net implementations** - GitHub / PapersWithCode

---

## 12. Glosario Rápido

| Término                  | Significado                                 |
| ------------------------ | ------------------------------------------- |
| **Cover**                | Archivo portador original (limpio)          |
| **Stego / Stego-objeto** | Archivo con payload oculto                  |
| **Payload**              | Información secreta embebida                |
| **Embedding**            | Proceso de ocultar                          |
| **Extraction**           | Proceso de recuperar                        |
| **Stegoanálisis**        | Detección/análisis de estego                |
| **Chi-square / χ²**      | Test estadístico uniformidad LSB            |
| **RS Analysis**          | Test grupos regulares/singulares (Fridrich) |
| **SPA**                  | Sample Pair Analysis                        |
| **DCT**                  | Discrete Cosine Transform (JPEG)            |
| **Matrix Embedding**     | Codificación eficiente (síndrome, Hamming)  |
| **LSB Matching / ±1**    | Embedding ±1 en vez de forzar bit           |

---

*Etiquetas: #estegoanalisis #deteccion #forense #chi-square #rs-analysis #spam #srnet #stegdetect #stegspy #blue-team #contramedidas*