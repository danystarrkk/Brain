---
aliases:
  - Steghide
  - Steghide Flags
  - Ocultar Datos con Steghide
tags:
  - sec/stego
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Stegseek - Guía Rápida]]"
  - "[[Esteganografía - Conceptos, Herramientas y Práctica]]"
  - "[[Estegoanálisis - Detección y ContraMedidas]]"
  - "[[Zsteg - Análisis LSB PNG]]"
fuente: "http://steghide.sourceforge.net/documentation/"
---

# Steghide — Uso Avanzado

**steghide** es la herramienta CLI de esteganografía más usada en CTF y auditorías: embebe un archivo cifrado (AES-256/Rijndael por defecto) dentro de una **cover JPEG, BMP, WAV o AU** protegido por una passphrase. Proyecto C maduro (2003), `apt install steghide`.

> [!info] Cómo funciona por dentro (por qué se ve igual y qué deja de rastro)
> 1. Comprime el payload con **zlib** (nivel `-z`).
> 2. Lo cifra con **Rijndael-256 en CBC** y IV aleatorio (o Serpent/Twofish si se pide).
> 3. Deriva una clave de la passphrase con un esquema basado en **SHA-1** y escribe los datos en los **componentes de baja frecuencia de los coeficientes DCT** de la cover (no en píxeles crudos: por eso sobrevive ligeramente mejor que el LSB plano y **no funciona en PNG**, que es sin pérdidas y no tiene DCT).
> 4. Guarda un estándar de datos con nombre del archivo, algoritmo, compresión y **checksum SHA-1** del payload (salvo `-N`).
>
> Ese primer byte/estructura con la cadena `steghide` es justamente la **firma** que usan los detectores: ver [[Estegoanálisis - Detección y ContraMedidas]] §3.

## Instalación y verificación

```bash
sudo apt update && sudo apt install steghide      # Debian/Ubuntu/Kali
sudo pacman -S steghide                            # Arch
steghide --version                                 # steghide 0.5.1
steghide --help | grep -A 20 "encryption algorithms"
```

## Tabla completa de flags

| Flag | Comando | Significado |
|---|---|---|
| `-cf, --coverfile FILE` | embed | Archivo portador (obligatorio en embed) |
| `-ef, --embedfile FILE` | embed | Archivo a ocultar (obligatorio en embed) |
| `-sf, --stegofile FILE` | extract/info | Archivo estego a analizar (obligatorio) |
| `-xf, --extractfile FILE` | extract | Ruta de salida del payload (defecto: nombre original guardado) |
| `-p, --passphrase STR` | embed/extract | Passphrase en claro; **si se omite, se pide en modo interactivo** |
| `-K, --keyfile FILE` | embed/extract | Usa un keyfile binario de 32 bytes en lugar de passphrase |
| `-z, --compress N` | embed | Compresión zlib 0–9 (defecto **9**; `0` = sin comprimir) |
| `-e, --encrypt ALGO` | embed | `rijndael-256` (defecto), `rijndael-128`, `des`, `twofish-256`, `serpent-256`, `cast-128`… |
| `-N, --dontembedname` | embed | No guarda el nombre original del payload (ahorra bytes, rompe el checksum) |
| `-f, --force` | embed/extract | Sobrescribe archivos existentes sin avisar |
| `-q, --quiet` | todos | Salida silenciosa (scripts) |
| `-v, --verbose` | todos | Salida detallada |
| `-h, --help` / `-V` | todos | Ayuda / versión |

## Los tres subcomandos

### 1. `info` — diagnóstico sin extraer

```bash
steghide info sospechoso.jpg
# Si NO hay datos ocultos: "steghide 0.5.1 - input file ... does not have embedded data"
#   → no es steghide (probá zsteg/binwalk/strings)
# Si SÍ hay: pide passphrase y, con la correcta, muestra:
#   embedded file: "secreto.txt" (1024 bytes)
#   embedded bits: 8192 (1.0 KB)
#   encryption: rijndael-256   compression: 9
```

> [!tip] Señal de estego sin conocer la passphrase
> Que `info` **pida passphrase** ya confirma datos embebidos: con archivos limpios no la pide. Guardá esa señal antes de correr `stegseek` (ver [[Stegseek - Guía Rápida]]).

### 2. `embed` — ocultar

```bash
steghide embed -cf <cover> -ef <payload> [OPCIONES]
```

| Escenario | Comando |
|---|---|
| Interactivo (no deja passphrase en historial) | `steghide embed -cf foto.jpg -ef secreto.txt` |
| Automatizado | `steghide embed -cf foto.jpg -ef secreto.txt -p "$STEGHIDE_PASS" -q` |
| Máxima capacidad (payload ya comprimido/cifrado) | `steghide embed -cf cover.bmp -ef payload.enc -z 0 -N -f` |
| Keyfile en vez de passphrase | `openssl rand -hex 32 > key.bin` · `steghide embed -cf cover.jpg -ef p.enc -K key.bin -q` |
| Algoritmo alternativo | `steghide embed -cf cover.jpg -ef p.enc -p "$PASS" -e serpent-256` |

### 3. `extract` — extraer

```bash
steghide extract -sf <stego> [-xf salida] (-p PASS | -K key.bin) [-f] [-q]

# Ejemplos
steghide extract -sf stego.jpg                      # interactivo
steghide extract -sf stego.jpg -xf payload.enc -p "$PASS" -f -q
steghide extract -sf stego.jpg -K key.bin -q
```

Mensajes típicos:

| Salida | Significado | Acción |
|---|---|---|
| `wrote extracted data to "secreto.txt".` | Éxito | `file` + `strings` + recursión |
| `could not extract ... wrong passphrase?` | Passphrase incorrecta | `stegseek --crack` |
| `steghide: could not open ...` | Ruta/permisos | Revisar path |
| `unsupported file format` | PNG/GIF/MP3/… | Convertir o usar otra técnica |
| `passphrase is too short` | Política de longitud | Passphrase ≥ 12 caracteres |
| `cover file too small` | Payload > capacidad | Ver cálculo siguiente |
| `checksum mismatch` | Payload corrupto o `-N` | Re-embeber sin `-N` |

## Capacidad: cuánto cabe

```bash
steghide info cover.jpg     # capacity: 12.3 KB (12,648 bytes)  ← valor real, sin modificar
```

**Estimación teórica** (LSB-like sobre componentes de baja frecuencia):

```
capacidad ≈ (ancho × alto × 3 canales × 0,5 bits) / 8 × ~0,7 (overhead)
800×600 → 72.000 B teóricos → ~12 KB reales tras cifrado+zlib
```

| Cover | Capacidad relativa | Comentario |
|---|---|---|
| JPEG pequeño (< 50 KB) | Poca (1–5 KB) | Fotos con mucha compresión |
| JPEG 800×600+ | 10–40 KB | Suficiente para texto/archivos pequeños |
| BMP (sin comprimir) | **≈ 12,5% del archivo** | Cover ideal: GB de capacidad |
| WAV 44,1 kHz estéreo | Alto (≈ 1 bit/muestra × canales) | Audio sin comprimir |
| PNG / GIF / MP3 | **0 — no soportado** | steghide no los acepta |

> [!warning] La capacidad es del portador, no del payload cifrado
> El payload pasa por zlib **y luego** por AES: los datos ya cifrados **no comprimen**. Si embeddes un `.enc`, `-z 9` no ayuda casi nada; si embeddes texto plano, la compresión puede multiplicar la capacidad por 3–5×.

## Modos de passphrase

| Modo | Comando | Ventaja | Riesgo |
|---|---|---|---|
| **Interactivo** | omitir `-p` | No deja rastro en `history`/`ps` | Incomodo en scripts |
| **Flag `-p`** | `-p "MiPass123!"` | Automático | **Aparece en `ps` e historial** |
| **Variable de entorno** | `-p "$PASS"` | No hardcodea; se limpia con `unset` | Visible en `/proc/<pid>/environ` si hay root |
| **Keyfile `-K`** | `-K key.bin` (32 bytes) | 256 bits de entropía real, imposible de crackear | Pérdida del archivo = pérdida del payload |

```bash
# Flujo recomendado en scripts
export STEGHIDE_PASS="$(openssl rand -base64 32)"
steghide embed -cf foto.jpg -ef payload.enc -p "$STEGHIDE_PASS" -q
unset STEGHIDE_PASS
```

## Spoofing de extensión y poliglotas

steghide valida el **formato real**, no la extensión: puedes guardar el estego con una extensión engañosa y añadir material encima/debajo:

```bash
# 1) Estego con extensión cambiada
steghide embed -cf foto.jpg -ef payload.enc -p "$PASS" -q
mv foto.jpg nota.txt                # el archivo sigue siendo JPEG válido internamente

# 2) Añadir una ZIP al final (polyglot JPEG+ZIP: sigue abriéndose como imagen)
cat foto.jpg anexos.zip > reporte.jpg
unzip -l reporte.jpg                # también es un ZIP válido

# 3) Extensión falsa del payload guardado (el nombre real queda dentro del estándar
#    salvo que uses -N, que evita guardar el nombre original)
```

Detección del truco (lado analista): `file reporte.jpg`, `binwalk reporte.jpg`, `unzip -l` y `steghide info` ignorando la extensión.

## Verificación de integridad visual

```bash
ls -lh cover.jpg stego.jpg                       # delta ≈ payload + overhead
compare -metric AE cover.jpg stego.jpg diff.png  # 0 píxeles distintos = imperceptible
convert cover.jpg stego.jpg -compose difference -composite \
        -evaluate pow 2 -scale 1x1! txt:-        # valor cercano a 0 = OK
```

## Flujos de trabajo

### Exfiltración en laboratorio (red team)

```bash
#!/bin/bash
# exfil.sh — preparar payload y ocultarlo en una cover legítima
PAYLOAD="datos_sensibles.tar.gz"; COVER="foto_equipo.jpg"; PASS="$(openssl rand -base64 32)"
tar -czf - "$PAYLOAD" | openssl enc -aes-256-cbc -pbkdf2 -k "$PASS" -out payload.enc
steghide embed -cf "$COVER" -ef payload.enc -p "$PASS" -z 9 -f -q
shred -u payload.enc                                 # limpiar el intermedio
echo "$PASS" | gpg --encrypt -r "analista@equipo" > pass.gpg   # passphrase fuera de línea
```

### Escaneo defensivo de un directorio

```bash
for f in *.jpg *.jpeg *.bmp *.wav *.au; do
  steghide info "$f" 2>/dev/null | grep -q "embedded" && \
    echo "[!] steghide detectado: $f" && steghide info "$f" 2>&1 | grep -E "embedded|encryption|compression"
done
```

### Preparar un challenge CTF

```bash
echo "flag{st3gh1d3_m4st3r}" > flag.txt
openssl enc -aes-256-cbc -pbkdf2 -k "pista_en_descripcion" -in flag.txt -out flag.enc
steghide embed -cf meme.jpg -ef flag.enc -p "pista_en_descripcion" -z 9 -f -q
# pista en la descripción del reto; si es débil → el equipo resolverá con stegseek
```

## Detección (blue team)

```bash
steghide info sospechoso.jpg 2>&1 | grep -q embedded && echo "STEGHIDE DETECTADO"   # pide pass ⇒ hay datos
stegdetect sospechoso.jpg                                                          # "steghide(??)"
xxd sospechoso.jpg | grep -i steghide                                              # firma a offset variable
ent sospechoso.jpg                                                                 # entropía alta en zonas planas
```

Contra-medidas del defensor: re-codificar imágenes en la ingesta (destruye el estego) o exigir formatos re-encoded; ver [[Estegoanálisis - Detección y ContraMedidas]] §6.2.

## Errores comunes

1. **Intentar `steghide` sobre PNG**: formatos soportados = JPEG, BMP, WAV, AU. Para PNG → [[Zsteg - Análisis LSB PNG]].
2. **Esperar que `-z 9` ayude con payloads cifrados**: AES no comprime; comprime **antes** de cifrar.
3. **Confundir `info` sin passphrase con "no hay datos"**: si pide passphrase, **hay** datos.
4. **Usar `-N` por ahorro de bytes** y perder la capacidad de saber el nombre/tipo del payload.
5. **Passphrase corta o de diccionario**: `stegseek` la rompe en segundos (ver [[Stegseek - Guía Rápida]]).
6. **Sobrescribir la cover sin `-f`** y abortar a mitad de operación: steghide puede dejar el archivo a medias.
7. **No verificar la extracción tras embed**: siempre hacer `steghide info` + `extract` de prueba.

## Alternativas modernas

| Herramienta | Ventaja frente a steghide | Formatos |
|---|---|---|
| **openstego** | Múltiples algoritmos + GUI (Java) | PNG, JPEG |
| **stegonline** | Web, LSB interactivo, sin instalar | PNG, JPEG |
| **stegcloak** | Texto (zero-width Unicode) | texto |
| **outguess / f5** | DCT con preservación estadística | JPEG |
| **Script Python propio** | Sin firma de herramienta conocida | cualquiera |

## Cheatsheet

```bash
steghide info FILE                       # ¿hay datos? (pide passphrase)
steghide embed -cf cover.jpg -ef data -p "$PASS" -z 9 -f -q
steghide extract -sf stego.jpg -xf out -p "$PASS" -f -q
steghide extract -sf stego.jpg -K key.bin -q
stegseek --crack stego.jpg rockyou.txt   # passphrase olvidada/débil
xxd stego.jpg | grep -i steghide         # firma
```

## Referencias

- Manual oficial: <http://steghide.sourceforge.net/documentation/>
- Hetzl, S. — *Steghide: A Steganography Program* (2003)
- `man steghide`
- Stegseek (cracking): <https://github.com/RickdeJager/stegseek>

## Relacionadas

- [[Stegseek - Guía Rápida]] — cracking de passphrases
- [[Esteganografía - Conceptos, Herramientas y Práctica]] — flujos completos y taxonomy
- [[Estegoanálisis - Detección y ContraMedidas]] — cómo detectan steghide
- [[Zsteg - Análisis LSB PNG]] — cuando el formato es PNG
