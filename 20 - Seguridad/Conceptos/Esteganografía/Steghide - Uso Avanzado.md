# Steghide - Uso Avanzado

> **Herramienta**: Esteganografía CLI madura (C, 2003-presente). Embebe datos cifrados (AES-256/Rijndael-256) en JPEG, BMP, WAV, AU con passphrase.
> **Proyecto**: http://steghide.sourceforge.net/ | `apt install steghide`

---

## Conceptos Clave

| Concepto | Detalle |
|----------|---------|
| **Formato propietario** | Magic bytes `steghide` + header con metadatos cifrados |
| **Cifrado por defecto** | Rijndael-256 (AES-256) en modo CBC con IV aleatorio |
| **Compresión** | zlib (nivel 1-9) antes de cifrar + ocultar |
| **Integridad** | Checksum SHA-1 del payload original (opcional `-N`) |
| **Capacidad real** | ~10-15% del tamaño cover (JPEG), más en BMP/WAV |

---

## Instalación y Verificación

```bash
# Debian/Ubuntu/Kali
sudo apt update && sudo apt install steghide

# Arch
sudo pacman -S steghide

# Verificar
steghide --version
# steghide 0.5.1

# Ver algoritmos soportados
steghide --help | grep -A 20 "encryption algorithms"
```

---

## Capacidad Estimada por Cover

```bash
# Ver capacidad exacta sin modificar
steghide info cover.jpg
# "steghide 0.5.1 - 800x600 JPEG image
#  capacity: 12.3 KB (12,648 bytes)"

# Fórmula aproximada JPEG:
# Capacidad ≈ (Ancho × Alto × 3 canales × 0.5 bits/LSB) / 8 × 0.7 (overhead)
# 800×600 ≈ 108 KB teórico → ~12 KB real
```

---

## Embedding (Ocultar Datos) - Guía Completa

### Sintaxis Completa

```bash
steghide embed -cf <cover> -ef <embed_file> [OPCIONES]

# Opciones principales:
# -cf, --coverfile       Archivo portador (requerido)
# -ef, --embedfile       Archivo a ocultar (requerido)
# -p, --passphrase       Passphrase (si omitido, pide interactivamente)
# -z, --compress 1-9     Nivel compresión zlib (default: 9, 0 = sin compresión)
# -e, --encrypt ALGO     Algoritmo: rijndael-256, serpent-256, twofish-256, etc.
# -K, --key FILE         Usar archivo clave binaria (32 bytes) en vez de passphrase
# -N, --dontembedname    No guardar nombre archivo original (ahorra ~bytes, menos integridad)
# -f, --force            Sobrescribir cover original (sin backup)
# -q, --quiet            Modo silencioso
```

### Ejemplos Progresivos

#### 1. Básico (interactivo)
```bash
steghide embed -cf foto.jpg -ef secreto.txt
# Enter passphrase: ********
# Re-Enter passphrase: ********
# embedding "secreto.txt" in "foto.jpg"... done
```

#### 2. Automatizado (scripts)
```bash
# Passphrase por variable de entorno (más seguro que CLI)
export STEGHIDE_PASS="MiPassFuerte2024!"
steghide embed -cf foto.jpg -ef secreto.txt -p "$STEGHIDE_PASS" -q
unset STEGHIDE_PASS
```

#### 3. Máxima capacidad (sin checksum, sin compresión)
```bash
steghide embed -cf cover.bmp -ef payload.bin -p "pass" -z 0 -N -f
# -z 0: sin compresión (payload ya comprimido/cifrado)
# -N: no guarda nombre archivo (pierde integridad, gana ~50 bytes)
# -f: sobrescribe cover original
```

#### 4. Clave binaria (keyfile) en vez de passphrase
```bash
# Generar clave 32 bytes (256 bits)
openssl rand -hex 32 > key.bin
# O desde passphrase: echo -n "mipass" | sha256sum | cut -d' ' -f1 | xxd -r -p > key.bin

steghide embed -cf cover.jpg -ef payload.enc -K key.bin -q
```

#### 5. Algoritmo alternativo (Serpent/Twofish)
```bash
steghide embed -cf cover.jpg -ef payload.enc -p "pass" -e serpent-256
# Ver todos: steghide --help | grep -A 30 "encryption algorithms"
```

---

## Extraction (Extraer Datos)

### Sintaxis

```bash
steghide extract -sf <stego_file> [OPCIONES]

# Opciones:
# -sf, --stegofile       Archivo con datos ocultos
# -xf, --extractfile     Archivo salida (default: nombre original guardado)
# -p, --passphrase       Passphrase
# -K, --key FILE         Archivo clave binaria
# -f, --force            Sobrescribir salida si existe
# -q, --quiet            Silencioso
```

### Ejemplos

```bash
# 1. Interactivo
steghide extract -sf stego.jpg
# Enter passphrase: ********
# wrote extracted data to "secreto.txt".

# 2. Automatizado con salida personalizada
steghide extract -sf stego.jpg -xf payload_dec.enc -p "$PASS" -f -q

# 3. Con keyfile
steghide extract -sf stego.jpg -K key.bin -q
```

---

## Info / Análisis sin Extraer

```bash
# Info básica (requiere passphrase si hay datos)
steghide info stego.jpg
# "steghide 0.5.1 - 800x600 JPEG image
#  embedded file: "secreto.txt" (1024 bytes)
#  encryption: rijndael-256
#  compression: 9"

# Info SIN passphrase (solo detecta si hay datos steghide)
steghide info stego.jpg 2>&1 | head -5
# Si NO pide passphrase → no hay datos steghide o cover vacía
```

---

## Verificar Integridad del Cover

```bash
# Comparar cover original vs stego (deben ser visualmente idénticos)
# 1. Hash perceptual (diferencia visual)
compare -metric AE cover.jpg stego.jpg diff.png 2>&1
# 0 = idénticos visualmente

# 2. Diferencia pixel a pixel (ImageMagick)
convert cover.jpg stego.jpg -compose difference -composite -evaluate pow 2 -scale 1x1! txt:-
# Valor cercano a 0 = imperceptible

# 3. Tamaño archivo
ls -lh cover.jpg stego.jpg
# Diferencia ≈ tamaño payload + overhead (~100-500 bytes)
```

---

## Flujos de Trabajo Comunes

### Flujo 1: Exfiltración Segura (Red Team)
```bash
#!/bin/bash
# exfil.sh - Preparar payload y ocultar en cover legítima

PAYLOAD="datos_sensibles.tar.gz"
COVER="foto_perfil_equipo.jpg"        # Cover legítima del servidor web
STEGO="foto_perfil_equipo_v2.jpg"     # Nombre plausible
PASS="$(openssl rand -base64 32)"     # Passphrase fuerte aleatoria

# 1. Cifrar payload
tar -czf - "$PAYLOAD" | openssl enc -aes-256-cbc -pbkdf2 -k "$PASS" -out payload.enc

# 2. Embed
steghide embed -cf "$COVER" -ef payload.enc -p "$PASS" -z 9 -f

# 3. Limpiar rastros
shred -u payload.enc
# Subir stego.jpg al servidor (reemplaza original)

# 4. Guardar passphrase en vault/gestor seguro
echo "$PASS" | gpg --encrypt -r "analista@equipo" > pass.gpg
```

### Flujo 2: CTF - Preparar Challenge
```bash
#!/bin/bash
# create_challenge.sh

FLAG="flag{st3gh1d3_m4st3r_2024}"
COVER="meme.jpg"
OUT="challenge.jpg"

echo "$FLAG" > flag.txt
openssl enc -aes-256-cbc -pbkdf2 -k "pista_en_descripcion" -in flag.txt -out flag.enc
steghide embed -cf "$COVER" -ef flag.enc -p "pista_en_descripcion" -z 9 -f -q
rm flag.txt flag.enc

echo "Challenge listo: $OUT"
echo "Pista: La passphrase está en la descripción del reto"
```

### Flujo 3: Verificación Rápida (Blue Team)
```bash
#!/bin/bash
# scan_steghide.sh - Escanear directorio buscando steghide

for f in *.jpg *.jpeg *.bmp *.wav *.au; do
    if steghide info "$f" 2>/dev/null | grep -q "embedded file"; then
        echo "[!] Steghide detectado: $f"
        steghide info "$f" 2>/dev/null | grep -E "embedded file|encryption|compression"
    fi
done
```

---

## OPSEC y Buenas Prácticas

| Práctica | Por qué |
|----------|---------|
| **Cover única por operación** | Evita análisis diferencial (comparar covers) |
| **Cover con alta entropía** | Fotos RAW, ruido natural, no capturas de pantalla |
| **Passphrase > 20 chars, aleatoria** | Resiste stegseek/rockyou |
| **Cifrar ANTES (AES-256)** | Defensa en profundidad |
| **Comprimir payload (zlib 9)** | Reduce tamaño, aumenta capacidad efectiva |
| **Limpiar metadata EXIF** | `exiftool -all= cover.jpg` antes de embed |
| **Verificar extracción ANTES de enviar** | Evita corrupción silenciosa |
| **No reutilizar cover original pública** | Análisis forense detecta modificación |

---

## Detección (Blue Team)

```bash
# 1. steghide info (rápido)
steghide info sospechoso.jpg 2>&1 | grep -q "embedded" && echo "STEGHIDE DETECTADO"

# 2. stegdetect (JPEG)
stegdetect sospechoso.jpg
# "sospechoso.jpg : steghide(??)"

# 3. Firmas hex (magic bytes steghide)
xxd sospechoso.jpg | grep -i "steghide"
# Busca en últimos bytes del archivo

# 4. Entropía anómala
ent sospechoso.jpg
# Entropía > 7.9 en áreas planas = sospechoso
```

---

## Troubleshooting

| Error | Causa | Solución |
|-------|-------|----------|
| "could not open cover file" | Ruta incorrecta / permisos | Verificar path, `chmod 644` |
| "passphrase is too short" | < 12 chars (policy) | Usar passphrase > 12 chars |
| "cover file too small" | Payload > capacidad | Cover mayor o payload menor |
| "wrong passphrase" | Passphrase incorrecta | stegseek + wordlist |
| "checksum mismatch" | Datos corruptos / -N usado | Re-embedding sin -N |
| "unsupported format" | PNG, GIF, MP3, etc. | Convertir a JPEG/BMP/WAV |

---

## Alternativas Modernas

| Herramienta | Ventaja vs steghide |
|-------------|---------------------|
| **openstego** | Plugins múltiples algoritmos, GUI, Java |
| **stegonline** | Web, LSB visual, sin instalar |
| **stegcloak** | Texto (Unicode zero-width), no binario |
| **deepsteg** | Adversarial learning, indetectable estadísticamente |
| **Custom Python** | Control total, sin firmas conocidas |

---

## Referencias

- `man steghide`
- http://steghide.sourceforge.net/documentation/
- Source: https://github.com/StefanoDeVuono/steghide (fork mantenido)
- Paper: "Steghide - A Steganography Program" - Hetzl (2003)

---

*Etiquetas: #steghide #embedding #extraction #aes256 #rijndael #opsec #redteam #ctf*