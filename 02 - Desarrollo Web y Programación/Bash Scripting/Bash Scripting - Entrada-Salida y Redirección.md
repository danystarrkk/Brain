---
title: "Bash Scripting - Entrada/Salida y Redirección"
tags:
  - lang/bash
  - concept/io
  - concept/redirection
  - concept/file-descriptors
  - level/intermedio
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Entrada/Salida y Redirección

> **Objetivo**: Dominar stdin/stdout/stderr, descriptores de archivo, here-docs, y redirecciones avanzadas.

---

## 1. Descriptores de Archivo Estándar

| FD | Nombre | Descripción | Default |
|----|--------|-------------|---------|
| 0 | `stdin` | Entrada estándar | Teclado / Pipe entrada |
| 1 | `stdout` | Salida estándar | Pantalla / Pipe salida |
| 2 | `stderr` | Error estándar | Pantalla (no bufferizado) |

```bash
# Referencias equivalentes
comando           # stdout → FD 1, stderr → FD 2
comando 1>out     # Redirige stdout
comando 2>err     # Redirige stderr
comando >out 2>err # Ambos por separado
```

---

## 2. Redirecciones Básicas

### 2.1 Salida (Output)

```bash
# Sobrescribir (truncar)
comando > archivo.txt        # stdout → archivo
comando 2> archivo.txt       # stderr → archivo
comando > out.txt 2> err.txt # Separados

# Append (añadir al final)
comando >> archivo.txt       # stdout append
comando 2>> archivo.txt      # stderr append

# Combinar stdout y stderr
comando > archivo.txt 2>&1   # Clásico: stdout→archivo, stderr→stdout
comando &> archivo.txt       # Bash shortcut (equivale a anterior)
comando &>> archivo.txt      # Append ambos
```

### 2.2 Entrada (Input)

```bash
# Desde archivo
comando < archivo.txt        # stdin ← archivo

# Here-string (Bash, Zsh, Ksh)
comando <<< "string"         # stdin ← string literal
comando <<< "$variable"      # stdin ← contenido variable

# Here-document (ver sección 4)
comando << 'EOF'
contenido multilínea
EOF
```

---

## 3. Redirección de Descriptores Personalizados (FD 3-9)

### 3.1 Abrir/Cerrar Descriptores con `exec`

```bash
# Abrir archivo en FD 3 para lectura/escritura
exec 3<> archivo.txt

# Abrir solo lectura
exec 3< archivo.txt

# Abrir solo escritura (crear/truncar)
exec 3> archivo.txt

# Abrir append
exec 3>> archivo.txt

# Cerrar descriptor
exec 3>&-   # Cerrar FD 3 (output)
exec 3<&-   # Cerrar FD 3 (input)
exec 3<&-   # Cerrar FD 3 (ambos)

# Duplicar descriptor
exec 4>&3   # FD 4 apunta a mismo lugar que FD 3
exec 4<&3   # Duplicar input
```

### 3.2 Usar Descriptores Personalizados

```bash
# Escribir a FD 3
echo "Log interno" >&3

# Leer de FD 3
read -u 3 linea

# Redirigir comando a FD 3
comando >&3

# Copiar FD 1 (stdout) a FD 3, luego restaurar
exec 3>&1           # FD 3 = stdout actual
exec 1> log.txt     # stdout → log.txt
echo "Esto va al log"
exec 1>&3           # Restaurar stdout desde FD 3
exec 3>&-           # Cerrar FD 3
echo "Esto va a pantalla"
```

---

## 4. Here-Documents y Here-Strings

### 4.1 Here-Document (`<<`)

```bash
# Básico (con expansión de variables)
cat << EOF
Hola $USER
Hoy es $(date)
EOF

# Sin expansión (comillas en delimitador)
cat << 'EOF'
Hola $USER          # Literal: $USER
No se expande $(date)
EOF

# Con tabulador suprimido (<<-)
if true; then
    cat <<- EOF
	Línea 1 (tab al inicio ignorado)
	    Línea 2 indentada
	EOF
fi

# Heredoc a variable (Bash 4+)
read -r -d '' variable << 'EOF'
Contenido
multilínea
EOF
echo "$variable"
```

### 4.2 Here-String (`<<<`)

```bash
# Equivalente a: echo "string" | comando
tr 'a-z' 'A-Z' <<< "hola mundo"  # HOLA MUNDO

# Múltiples líneas
cat <<< $'línea1\nlínea2\nlínea3'

# Útil para funciones que leen stdin
procesar() { while read -r line; do echo ">$line"; done; }
procesar <<< "datos de prueba"
```

---

## 5. Tee - Duplicar Salida

```bash
# stdout → archivo Y pantalla
comando | tee archivo.log

# Append
comando | tee -a archivo.log

# Múltiples archivos
comando | tee archivo1.log archivo2.log

# stderr también (Bash 4+)
comando 2>&1 | tee archivo.log

# Con process substitution para archivos separados
comando > >(tee stdout.log) 2> >(tee stderr.log >&2)
```

---

## 6. Process Substitution (`<()` y `>()`)

> **Potente: trata salida/entrada de comando como archivo temporal**

### 6.1 `<(comando)` - Output como Archivo de Entrada

```bash
# Comparar salida de dos comandos sin archivos temporales
diff <(comando1) <(comando2)

# Leer salida de comando en while (evita subshell)
while IFS= read -r linea; do
    echo "Procesado: $linea"
done < <(comando_que_genera_salida)

# Múltiples inputs simultáneos
paste <(cmd1) <(cmd2) <(cmd3)
```

### 6.2 `>(comando)` - Input como Archivo de Salida

```bash
# Enviar stdout a un comando y stderr a otro
comando > >(tee stdout.log) 2> >(tee stderr.log >&2)

# Log rotativo simple
exec > >(rotatelogs /var/log/app/%Y-%m-%d.log 86400)
```

---

## 7. Redirección Avanzada y Patrones

### 7.1 Swap stdout/stderr

```bash
# Intercambiar: 3=stdout, 1=stderr, 2=stdout
comando 3>&1 1>&2 2>&3 3>&-

# Uso: capturar stderr en variable, stdout a pantalla
error=$(comando 3>&1 1>&2 2>&3 3>&-)
```

### 7.2 Silenciar Selectivamente

```bash
# Solo errores
comando >/dev/null

# Solo salida normal
comando 2>/dev/null

# Nada (ambos)
comando >/dev/null 2>&1
comando &>/dev/null
```

### 7.3 Logging Estructurado

```bash
# Setup logging al inicio del script
setup_logging() {
    local log_file="${1:-/tmp/script_$(date +%s).log}"
    exec 3>&1                    # FD 3 = stdout original
    exec 4>&2                    # FD 4 = stderr original
    exec 1> >(tee -a "$log_file")   # stdout → tee → log + FD3
    exec 2> >(tee -a "$log_file" >&2) # stderr → tee → log + FD4
}

# Restaurar
restore_logging() {
    exec 1>&3 3>&-   # Restaurar stdout, cerrar FD3
    exec 2>&4 4>&-   # Restaurar stderr, cerrar FD4
}

trap restore_logging EXIT
setup_logging "/var/log/mi_script.log"
```

---

## 8. Lectura/Escritura de Archivos Eficiente

### 8.1 `mapfile` / `readarray` (Bash 4+)

```bash
# Leer archivo completo a array (rápido)
mapfile -t lineas < archivo.txt
# -t: quita newline final

# Leer N líneas
mapfile -t -n 10 lineas < archivo.txt

# Desde comando
mapfile -t salida < <(comando)

# Callback por línea (Bash 4.4+)
mapfile -t -C 'procesar_linea' -c 1 < archivo.txt
```

### 8.2 Lectura Binaria / Raw

```bash
# Leer N bytes exactos
read -r -n 100 -d '' datos < archivo.bin

# Leer con timeout
read -r -t 5 -n 1 char
```

### 8.3 Escritura Atómica (Evitar archivos parciales)

```bash
escribir_atomico() {
    local archivo="$1" contenido="$2"
    local temp="${archivo}.tmp.$$"
    
    printf '%s' "$contenido" > "$temp" && mv "$temp" "$archivo"
    # mv es atómico en mismo filesystem
}
```

---

## 9. Pipes y FIFOs (Named Pipes)

### 9.1 Pipes Anónimos (Estándar)

```bash
# Pipeline simple
comando1 | comando2 | comando3

# Pipe falla si cualquier comando falla (con pipefail)
set -o pipefail
comando1 | comando2  # $? = exit code del último que falló
```

### 9.2 Named Pipes (FIFOs) - Comunicación Inter-Procesos

```bash
# Crear FIFO
mkfifo /tmp/mi_fifo

# Proceso escritor (bloquea hasta que haya lector)
echo "datos" > /tmp/mi_fifo &

# Proceso lector
read linea < /tmp/mi_fifo
echo "Recibido: $linea"

# Limpiar
rm /tmp/mi_fifo
```

---

## 10. `/dev/` Dispositivos Especiales

| Dispositivo | Uso |
|-------------|-----|
| `/dev/null` | Descarta todo (agujero negro) |
| `/dev/zero` | Produce bytes nulos (`\0`) infinitos |
| `/dev/random` | Entropía bloqueante (criptográfico) |
| `/dev/urandom` | Entropía no bloqueante (recomendado) |
| `/dev/stdin` | Referencia a FD 0 |
| `/dev/stdout` | Referencia a FD 1 |
| `/dev/stderr` | Referencia a FD 2 |
| `/dev/fd/N` | Referencia a descriptor N |
| `/dev/tcp/HOST/PORT` | Socket TCP (Bash built-in) |
| `/dev/udp/HOST/PORT` | Socket UDP (Bash built-in) |

```bash
# Ejemplos
dd if=/dev/zero of=archivo.bin bs=1M count=100  # Archivo 100MB ceros
head -c 32 /dev/urandom | base64  # Random string 32 bytes
exec 3<>/dev/tcp/google.com/80 && echo -e "GET / HTTP/1.0\n\n" >&3 && cat <&3  # HTTP raw
```

---

## 11. Buffering y `stdbuf` / `unbuffer`

```bash
# Forzar unbuffered (útil para pipes interactivos)
stdbuf -oL -eL comando  # Line-buffered stdout/stderr
stdbuf -o0 -e0 comando  # Unbuffered

# Con expect/unbuffer (para programas que detectan TTY)
unbuffer comando | tee log.txt
```

---

## 12. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Plantilla base
- [Procesos y Señales](./Bash%20Scripting%20-%20Procesos%20y%20Se%C3%B1ales.md) - Background jobs, `disown`, `nohup`
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - `while read` patterns
- [Debugging y Buenas Prácticas](./Bash%20Scripting%20-%20Debugging,%20Buenas%20Pr%C3%A1cticas%20y%20ShellCheck.md)

---

*Etiquetas: #bash #io #redireccion #file-descriptors #here-doc #here-string #tee #process-substitution #fifo #dev-null*