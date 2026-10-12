---
aliases: ["GHex", "Editor Hexadecimal", "Editor Hex", "ghex"]
tags: [tool/ghex]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Ghidra - Reverse Engineering Tool]]"
  - "[[Elevación de Privilegios mediante Explotación de Binarios]]"
  - "[[Privilegios SUID en Linux]]"
  - "[[Msfvenom - Herramienta de Generación de Payloads]]"
  - "[[Esteganografía - Conceptos, Herramientas y Práctica]]"
fuente: "https://wiki.gnome.org/Apps/Ghex"
---

# GHex: Editor Hexadecimal para Linux

**GHex** es el editor hexadecimal de GNOME: muestra un fichero como **offset | bytes en hex | representación ASCII** y permite editar byte a byte con precisión. En CTF y en análisis de binarios es la herramienta de "última milla": parchear un offset, cambiar un flag, corregir un byte de cabecera o leer la estructura interna de un formato cuando no hay —o no quieres— un descompilador.

> [!info] Por qué un editor hex y no un editor de texto
> Un fichero binario **no es texto**: `sed`/`vim` pueden transformar bytes (cambiar codificación, truncar en NULs, normalizar finales de línea) y dejarlo inservible. GHex trabaja sobre **bytes crudos**: cada carácter no imprimible tiene su casilla, ningún byte se "normaliza" y el tamaño del fichero bajo tu control. Para modificar un entero, un salto de máquina o los `magic bytes` de un formato, esto es lo correcto.

## Instalación

```bash
# Debian / Ubuntu / Kali
sudo apt update && sudo apt install ghex

# Fedora / RHEL
sudo dnf install ghex

# Lanzar
ghex fichero.bin
# o desde el menú de aplicaciones: "Hex Editor"
```

## Anatomía de la ventana

```text
  Offset     Hexadecimal (bytes)                        ASCII
  00000000   7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00   .ELF............
  00000010   03 00 3e 00 01 00 00 00  40 10 40 00 00 00 00 00   ..>.....@.@.....
```

| Panel | Qué muestra | Uso |
|---|---|---|
| **Offset** | Posición del byte desde el inicio (hex por defecto) | "¿En qué byte estoy?" — la referencia de todo |
| **Hex** | El byte en dos dígitos `00`-`FF` | Edición exacta y verificación de estructura |
| **ASCII** | Cada byte como carácter (no imprimible → `.`) | Localizar cadenas, rutas, URLs, flags |

Configuración habitual (menú View/Preferences): **bytes por línea** (16 es el estándar), **agrupación de dígitos** (1/2/4/8), **formato de dirección** (hex/decimal/octal) y mayúsculas/minúsculas.

> [!tip] Regla de oro del offset
> `offset 0x10 = decimal 16 = el décimo séptimo byte`. GHex muestra offsets en **hexadecimal**: si buscas en decimal con otra herramienta, convierte antes (`printf '%x\n' 276` → `0x114`). La mayoría de bugs de "no lo encuentra" son desajustes hex↔dec.

## Conceptos que necesitas antes de tocar nada

### Endianness (orden de bytes)

Un entero de 32 bits `0x01020304` se almacena:

| Sistema | Bytes en el fichero | Común en |
|---|---|---|
| **Little-endian** (LE) | `04 03 02 01` | x86/x64, ARM por defecto, **ELF/PE** |
| **Big-endian** (BE) | `01 02 03 04` | redes (TCP/IP), PowerPC, MIPS, Java `.class` |

GHex **no** interpreta: pone los bytes en orden de fichero. Si lees `40 10 40 00 00 00 00 00` en una cabecera ELF64 LE, el valor real es `0x0000000000401040`. Herramientas de comprobación:

```bash
xxd -e -l 32 fichero.bin      # dump en little-endian (interpretado)
xxd -l 32 fichero.bin         # dump en orden de fichero
```

### Magic bytes (cabeceras de formato)

| Formato | Bytes iniciales | ASCII |
|---|---|---|
| ELF | `7F 45 4C 46` | `.ELF` |
| PE/EXE (Windows) | `4D 5A 90 00` | `MZ` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG` |
| JPEG | `FF D8 FF E0` | `ÿØÿ` |
| GIF87a/89a | `47 49 46 38` | `GIF8` |
| ZIP/JAR/APK | `50 4B 03 04` | `PK..` |
| PDF | `25 50 44 46` | `%PDF` |
| gzip | `1F 8B 08` | — |

### Opcodes x86-64 que se parchean a diario

| Byte(s) | Instrucción | Uso en CTF |
|---|---|---|
| `90` | `NOP` | Rellenar/desactivar una comprobación |
| `EB xx` | `JMP corto` | Forzar salto incondicional |
| `E9 xx xx xx xx` | `JMP near` | Igual, a más distancia |
| `74 xx` / `75 xx` | `JE` / `JNE` | Cambiar `75→74` invierte la condición |
| `72`/`73`, `7E`/`7F` | `JB/JNB`, `JLE/JG` | Corregir comparaciones con signo |
| `31 C0` | `XOR EAX,EAX` | Dejar un registro a 0 |
| `C3` | `RET` | Salir antes (evitar un check) |
| `74 xx` → `EB xx` | `JE` → `JMP` | "Siempre pasa" el chequeo |

> [!warning] Tamaño y firmas
> **Nunca cambies el tamaño del fichero** al editar (insertar/borrar bytes desplaza todos los offsets siguientes y rompe el formato). Sustituye bytes por bytes equivalentes (`75 05` → `90 90`, manteniendo la longitud). Y ten en cuenta que parchear rompe **checksums y firmas**: binarios firmados, ZIP con CRC o saves con hash dejarán de validar.

## Flujo de trabajo en CTF (identificar → localizar → parchear → verificar)

### 1. Copia de seguridad + identificación

```bash
cp objetivo.bin objetivo.bin.bak        # siempre primero
file objetivo.bin                       # tipo detectado
xxd -l 64 objetivo.bin                  # primeros bytes en crudo
```

### 2. Localizar el offset (CLI, sin abrir GUI aún)

```bash
# Buscar una cadena y su offset exacto
grep -abo "flag{" objetivo.bin
grep -abo "admin" objetivo.bin

# Extraer cadenas legibles
strings -a -n 8 objetivo.bin | less

# Dump con offsets de una zona concreta
xxd -s 0x400 -l 128 objetivo.bin        # -s offset, -l longitud
```

`grep -abo` imprime `276:flag{...` → **276 (dec) = 0x114 (hex)**. Ese es el offset a abrir en GHex.

### 3. Abrir y editar en GHex

1. `ghex objetivo.bin`
2. Ir al offset: buscar con **Search → Find** (cadena o secuencia hex) o desplazarte con la barra de offset.
3. Modificar los bytes en el panel hexadecimal (escribir sobre ellos; el ASCII se actualiza).
4. **Guardar** (`Ctrl+S`) — GHex escribe *in place*, manteniendo la longitud.

### 4. Verificar el cambio

```bash
# Ver solo la zona modificada
xxd -s 0x114 -l 16 objetivo.bin

# Diff byte a byte contra la copia
cmp -l objetivo.bin objetivo.bin.bak

# Probar el binario parcheado
./objetivo.bin
```

## Recetas: los 5 usos reales

```bash
# 1. Ver cabecera y estructura inicial
xxd -l 128 fichero.bin | less
#    → identifica magic (7F454C46), clase (32/64 bits), endianness (offset 5: 01=LE)

# 2. Encontrar el offset de una cadena para editarla en GHex
grep -abo "TuClaveIncorrecta" binario
#    → 4821:TuClaveIncorrecta  →  abre GHex y ve a 0x12D5

# 3. Invertir una comprobación (JNE → JE) manteniendo tamaño
#    GHex: buscar el byte 75 de la rama, cambiarlo por 74 → guardar

# 4. Desactivar 3 instrucciones seguidas con NOPs
#    GHex: seleccionar los 3 bytes → escribir 90 90 90 → Ctrl+S
#    (equivale a: printf '\x90\x90\x90' | dd of=bin bs=1 seek=$((0x12D5)) conv=notrunc)

# 5. Parchear magic bytes (p.ej. que un .jpg se comporte como otro contenedor)
#    GHex: offset 0 → 89 50 4E 47 0D 0A 1A 0A → guardar (ojo: sin reestructurar el contenido)
```

**Por qué así:** los pasos 1-2 y 5 son CLI porque es más rápido y reproducible; la edición fina (3-4) es donde el GUI evita errores de escritura. `dd conv=notrunc` es la alternativa CLI exacta: escribe *sin truncar*.

## Errores comunes (mensaje → causa → arreglo)

1. **`ghex: open: Permission denied`** — sin permiso de lectura/escritura.
   → `chmod`/`chown` o lanzar con privilegios sobre una copia: `sudo ghex copia.bin`.
2. **`cannot execute binary file: Exec format error` tras editar** — corruptaste la cabecera (magic/offset mal).
   → Restaurar del `.bak` y volver a identificar con `xxd -l 16`.
3. **El cambio "no se guarda" o no se refleja** — guardaste en otra copia, o el programa sigue en memoria.
   → Verifica con `cmp -l a b` y **relanza** el binario tras parchear.
4. **Edité el offset equivocado** — mezcla hex/dec o copió mal `grep`.
   → Recalcula: `printf '%x' <dec>`; en GHex activa la columna de offset hexadecimal y confirma con `Search → Find`.
5. **El binario funciona pero "rompe" al final** — checksum/firma alterada.
   → O recalcular el hash (si el formato lo permite) o replantear el ataque (buscar el check en código en vez de en la firma).
6. **Inserté un byte y "todo el archivo cambió"** — los offsets siguientes se desplazaron.
   → Restaurar; edición de longitud fija únicamente.
7. **ZIP/PDF dañado tras parchear** — CRC/estructura por offsets.
   → Reparar desde la copia; para ZIP usa `zip -FF`/`binwalk` antes de tocar.
8. **Ficheros enormes van lentos** — GHex carga todo en memoria.
   → Extraer el trozo con `dd if=f of=part bs=1 skip=N count=M` y edita el fragmento; o usa herramientas de línea (`xxd -r`).

## Cheatsheet

### Atajos / menús de GHex

| Acción | Cómo |
|---|---|
| Abrir | `Ctrl+O` o File → Open |
| Guardar | `Ctrl+S` o File → Save |
| Guardar como | File → Save As |
| Copiar/pegar bytes | `Ctrl+C` / `Ctrl+V` sobre el panel hex |
| Buscar cadena o hex | Search → Find (`Ctrl+F`) |
| Siguiente coincidencia | Search → Find Next |
| Deshacer | `Ctrl+Z` (si tu versión lo soporta) |
| Cambiar bytes/línea, agrupación, formato de offset | View / Preferences |
| Cambiar panel a octal/binario/decimal | View → formato de visualización |

### Comandos CLI que acompañan a GHex

| Tarea | Comando |
|---|---|
| Ver cabecera | `xxd -l 64 f.bin` |
| Ver zona concreta | `xxd -s 0x114 -l 32 f.bin` |
| Dump little-endian | `xxd -e -l 32 f.bin` |
| Offset de una cadena | `grep -abo "texto" f.bin` |
| Cadenas largas | `strings -a -n 8 f.bin` |
| Tipo de fichero | `file f.bin` |
| Diff byte a byte | `cmp -l f.bin f.bak` |
| Parchear sin GUI | `printf '\x90\x90' \| dd of=f bs=1 seek=$((0x114)) conv=notrunc` |
| Revertir dump a binario | `xxd -r dump.hex f.bin` |
| Contar bytes | `wc -c f.bin` |

## Comparativa de editores hex

| Herramienta | Entorno | Fortaleza | Cuándo usarlo |
|---|---|---|---|
| **GHex** | GUI GNOME | Simple, integrado, suficiente para patch ligero | Edición puntual en Linux |
| `xxd`/`hexdump`/`od` | CLI | Rápido, scripteable, en todo sistema | Inspección y patch no interactivo |
| **Bless** | GUI GTK | Edición de ficheros grandes, multi-pestaña | Cuando GHex se queda corto |
| **ImHex** | GUI moderna | Patrones/estructuras (HScript), analizador CTF | CTFs de formatos complejos |
| **010 Editor** | Comercial | Templates binarios, forense | Entorno profesional |
| Python (`struct`) | Script | Parsing programático | Volumen/automatización |

## Detección y notas de seguridad

- Parchear un binario **no genera tráfico**: es una edición local. Lo que sí deja rastro es ejecutarlo (firmas EDR, CRC en logs, `integrity check` del software).
- En forense, la edición de un fichero altera timestamps y metadatos (`mtime`): usa copias y deja el original intacto para hash (`sha256sum` antes/después).
- Firma digital / secure boot: cualquier byte cambiado invalida la firma (esperado, no es un bug).

## Referencias

- [GHex en GNOME Wiki](https://wiki.gnome.org/Apps/Ghex) — documentación y descargas
- `man ghex` / `xxd -h`, `man hexdump` — ayuda local
- [ELF specification](https://refspecs.linuxfoundation.org/elf/elf.pdf) — estructura de cabeceras para saber qué offset leer
- GTFOBins — contexto de abuso de binarios (relacionado con [[Privilegios SUID en Linux]])

## Relacionadas

- [[Ghidra - Reverse Engineering Tool]] — cuando el byte a cambiar está en *código*, no en datos
- [[Elevación de Privilegios mediante Explotación de Binarios]] — entender qué estás parcheando
- [[Privilegios SUID en Linux]] — binarios SUID como objetivo de edición/abuso
- [[Msfvenom - Herramienta de Generación de Payloads]] — shellcode que a veces hay que incrustar a mano
- [[Esteganografía - Conceptos, Herramientas y Práctica]] — bytes ocultos en ficheros aparentemente inocuos
