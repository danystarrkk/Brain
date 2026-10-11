---
title: "Bash Scripting - Procesos y Señales"
tags:
  - lang/bash
  - concept/processes
  - concept/signals
  - concept/job-control
  - level/intermedio
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Procesos y Señales

> **Objetivo**: Gestionar procesos background, job control, señales, y comunicación inter-procesos.

---

## 1. Conceptos Básicos

### 1.1 Jerarquía de Procesos

```
shell (PID 1000)           ← Proceso padre / sesión líder
├── script.sh (PID 1001)   ← Hijo del shell
│   ├── sleep 10 (PID 1002)  ← Nieto
│   └── grep "foo" (PID 1003) ← Nieto
└── vim (PID 1004)         ← Otro hijo del shell
```

- **Padre**: Proceso que crea a otro (`fork()`)
- **Hijo**: Proceso creado, hereda FD, entorno, directorio actual
- **Huérfano**: Padre muere antes que hijo → adoptado por `init` (PID 1)
- **Zombie**: Hijo terminó pero padre no hizo `wait()` → entrada en tabla de procesos

### 1.2 Información de Procesos

```bash
$$           # PID del shell actual
$!           # PID del último proceso background
$BASHPID     # PID real del subshell actual (difiere de $$ en subshells)
$PPID        # PID del padre

# Info detallada
ps -p $$ -o pid,ppid,cmd
ps -ef | grep nombre
pgrep -f "patrón"     # PIDs por nombre/comando
pidof nombre          # PID por nombre exacto
```

---

## 2. Ejecución en Background

### 2.1 `&` - Background Simple

```bash
# Ejecutar en background
comando_largo &

# Capturar PID
comando_largo &
pid=$!
echo "Ejecutando en background con PID $pid"

# Ver jobs activos
jobs
jobs -l   # Con PIDs
jobs -p   # Solo PIDs

# Esperar a que termine
wait $pid
wait      # Espera TODOS los background jobs
```

### 2.2 `disown` - Desvincular de la Shell

```bash
# Por defecto: shell envía SIGHUP a hijos al cerrar
comando_largo &
disown      # Elimina el job más reciente de la tabla
disown %1   # Elimina job #1
disown -a   # Elimina todos
disown -h   # Marca para no recibir SIGHUP (no elimina de tabla)

# Combinado: background + disown en uno
comando_largo & disown
```

### 2.3 `nohup` - Inmune a SIGHUP

```bash
# Tradicional: nohup + redirección + background
nohup comando_largo > salida.log 2>&1 &

# nohup ignora SIGHUP y redirige stdout a nohup.out si no se especifica
```

---

## 3. Job Control (Control de Trabajos)

> **Requiere `set -m` (monitor mode) - activado por defecto en shells interactivos**

```bash
# En scripts, activar si se usa job control
set -m

# Suspender proceso foreground: Ctrl+Z
# Reanudar en background
bg %1       # Job #1 continúa en background
bg          # Job actual

# Traer a foreground
fg %1       # Job #1 a foreground
fg          # Job actual

# Matar job
kill %1     # Por número de job
kill -9 %1  # SIGKILL
```

### 3.1 Tabla de Jobs

```bash
jobs
# [1]  Running    sleep 100 &
# [2]- Stopped    vim archivo.txt
# [3]+ Stopped    cat > archivo

# Símbolos:
# +  Job actual (default para fg/bg)
# -  Job previo
```

---

## 4. Señales (Signals)

### 4.1 Señales Comunes

| Señal | Num | Nombre | Descripción | Default |
|-------|-----|--------|-------------|---------|
| `SIGHUP` | 1 | Hangup | Terminal cerrada | Terminar |
| `SIGINT` | 2 | Interrupt | Ctrl+C | Terminar |
| `SIGQUIT` | 3 | Quit | Ctrl+\ | Terminar + core dump |
| `SIGKILL` | 9 | Kill | Inmatable | Terminar FORZOSO |
| `SIGTERM` | 15 | Terminate | Petición educada | Terminar |
| `SIGSTOP` | 19 | Stop | Ctrl+Z (inmatable) | Pausar |
| `SIGCONT` | 18 | Continue | Reanudar tras STOP | Reanudar |
| `SIGUSR1` | 10 | User 1 | Usuario definido | Terminar |
| `SIGUSR2` | 12 | User 2 | Usuario definido | Terminar |

### 4.2 Enviar Señales

```bash
kill -TERM $pid      # Por nombre (recomendado)
kill -15 $pid        # Por número
kill -s TERM $pid    # Explícito
kill -9 $pid         # SIGKILL (último recurso)

# A todo un grupo de procesos
kill -TERM -$PGID    # Negativo = process group

# A todos los procesos del usuario
kill -TERM -1        # Cuidado!

# Por nombre (pkill)
pkill -TERM -f "patrón_comando"
pkill -TERM -u usuario
```

### 4.3 `trap` - Manejar Señales en Scripts

```bash
# Sintaxis: trap 'comandos' SEÑALES

# Limpiar al salir (EXIT = pseudo-señal al terminar script)
cleanup() {
    rm -f /tmp/temp_$$
    log_info "Limpieza completada"
}
trap cleanup EXIT

# Manejar Ctrl+C (SIGINT)
ctrl_c() {
    echo -e "\n[!] Interrumpido por usuario"
    exit 130  # 128 + 2 (SIGINT)
}
trap ctrl_c INT

# Manejar múltiples señales
manejar_seniales() {
    local senal=$1
    log_warn "Recibida señal: $senal"
    case $senal in
        TERM) log_info "Apagado ordenado..."; shutdown_graceful ;;
        USR1) log_info "Recarga config"; recargar_config ;;
        USR2) log_info "Dump estado"; dump_estado ;;
    esac
}
trap 'manejar_seniales TERM' TERM
trap 'manejar_seniales USR1' USR1
trap 'manejar_seniales USR2' USR2

# Ignorar señal
trap '' HUP   # Ignorar SIGHUP

# Resetear a default
trap - TERM   # Quita handler, restaura default
```

### 4.4 `trap` con Funciones y Argumentos

```bash
# Pasar señal como argumento
handler() {
    local sig=$1
    echo "Señal $sig recibida"
    # Limpieza específica
    [[ $sig == "TERM" ]] && guardar_estado
    exit 128 + $(kill -l "$sig")
}

trap 'handler TERM' TERM
trap 'handler INT' INT
trap 'handler QUIT' QUIT
```

---

## 5. Patrones de Procesos Robustos

### 5.1 Daemon Simple (Background + Disown + Logging)

```bash
daemonizar() {
    local cmd=("$@")
    local log_file="/var/log/${cmd[0]//\//_}.log"
    
    # Doble fork para desacoplar completamente
    (
        # Primer fork
        if (( $(pidof -x "${cmd[0]}" | wc -w) > 1 )); then
            echo "Ya existe instancia" >&2
            exit 1
        fi
        
        # Segundo fork (en subshell)
        (
            # Desvincular de terminal
            exec 0</dev/null
            exec 1>>"$log_file"
            exec 2>&1
            
            # Nuevo session leader
            setsid "${cmd[@]}"
        ) &
        disown
    ) &
}

# Uso
daemonizar mi_servidor --puerto 8080
```

### 5.2 Worker Pool (Procesos Paralelos Controlados)

```bash
# Procesar items en paralelo con N workers máximo
procesar_paralelo() {
    local max_jobs=4
    local -a pids=()
    
    for item in "$@"; do
        # Esperar si hay demasiados jobs
        while (( ${#pids[@]} >= max_jobs )); do
            # Reap hijos terminados
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[i]}" 2>/dev/null; then
                    wait "${pids[i]}" 2>/dev/null
                    unset 'pids[i]'
                fi
            done
            pids=("${pids[@]}")  # Reindexar
            sleep 0.1
        done
        
        # Lanzar worker
        procesar_item "$item" &
        pids+=($!)
    done
    
    # Esperar restantes
    for pid in "${pids[@]}"; do
        wait "$pid"
    done
}
```

### 5.3 Timeout con `timeout` (GNU coreutils)

```bash
# Ejecutar con límite de tiempo
timeout 10 comando_largo
# Exit codes: 124=timeout, 137=SIGKILL tras timeout

# Enviar señal específica tras timeout
timeout -s TERM 10 comando

# Mantener proceso si timeout (no matar)
timeout --preserve-status 10 comando
```

---

## 6. Comunicación Inter-Procesos (IPC)

### 6.1 Pipes Anónimos (Shell)

```bash
# Pipeline
productor | consumidor

# Bidireccional con coproc (Bash 4+)
coproc PRODUCTOR { productor; }
# ${PRODUCTOR[0]} = FD lectura, ${PRODUCTOR[1]} = FD escritura
echo "comando" >&"${PRODUCTOR[1]}"
read -u "${PRODUCTOR[0]}" respuesta
```

### 6.2 FIFOs (Named Pipes)

```bash
fifo="/tmp/mi_fifo_$$"
mkfifo "$fifo"
trap "rm -f $fifo" EXIT

# Escritor
{
    for i in {1..10}; do
        echo "Mensaje $i" > "$fifo"
        sleep 1
    done
} &

# Lector
while IFS= read -r msg < "$fifo"; do
    echo "Recibido: $msg"
done
```

### 6.3 Sockets Unix Domain (Bash 5+ con `socat` o `nc`)

```bash
# Servidor simple con socat
socat UNIX-LISTEN:/tmp/socket.sock,fork EXEC:"mi_handler.sh"

# Cliente
echo "petición" | socat - UNIX-CONNECT:/tmp/socket.sock
```

---

## 7. Sincronización

### 7.1 `wait` - Esperar PIDs Específicos

```bash
# Esperar PID específico
wait $pid

# Esperar todos los background jobs
wait

# Esperar con timeout (Bash 5.1+)
wait -n  # Espera el próximo que termine, retorna su PID
wait -n $pid1 $pid2  # Espera cualquiera de estos
```

### 7.2 Locks con `flock` (Archivo de Bloqueo)

```bash
# Lock exclusivo en sección crítica
(
    flock -x 200  # Bloquea FD 200
    # Sección crítica
    echo "Procesando..." >> /tmp/shared.log
) 200>/tmp/mi_lockfile

# No bloquear (fallar si ocupado)
flock -n 200 || { echo "Otro proceso ejecutándose"; exit 1; }
```

### 7.3 Semáforos con Directorio (Atómico)

```bash
# mkdir es atómico
adquirir_lock() {
    local lock_dir="/tmp/mi_lock_$$"
    if mkdir "$lock_dir" 2>/dev/null; then
        return 0
    fi
    return 1
}

liberar_lock() {
    rmdir "/tmp/mi_lock_$$"
}

if adquirir_lock; then
    # Sección crítica
    liberar_lock
fi
```

---

## 8. Subshells vs Procesos

```bash
# Subshell (mismo proceso, nuevo scope)
( comando1; comando2 )  # Padre espera, hereda FD

# Subshell en pipeline (bash: último en subshell)
comando1 | comando2  # comando2 en subshell

# Process substitution (subshell implícito)
while read line; do ...; done < <(comando)

# Diferencias clave:
# - Subshell: hereda variables, NO persisten cambios al padre
# - Background &: proceso separado, PID propio
# - Coproc: bidireccional, FDs accesibles
```

---

## 9. Debugging de Procesos

```bash
# Ver árbol de procesos
pstree -p $$
pstree -p $PID

# Strace (syscalls)
strace -p $PID -f -e trace=signal,process

# lsof (archivos abiertos)
lsof -p $PID

# /proc filesystem
cat /proc/$PID/status
cat /proc/$PID/fd/
ls -l /proc/$PID/fd/
```

---

## 10. Referencias Cruzadas

- [Entrada-Salida y Redirección](./Bash%20Scripting%20-%20Entrada-Salida%20y%20Redirecci%C3%B3n.md) - FDs, `exec`, pipes
- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - Bucles con background jobs
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md) - Funciones en background
- [Debugging y Buenas Prácticas](./Bash%20Scripting%20-%20Debugging,%20Buenas%20Pr%C3%A1cticas%20y%20ShellCheck.md)

---

*Etiquetas: #bash #procesos #señales #signals #background #disown #nohup #job-control #trap #ipc #flock #daemon*