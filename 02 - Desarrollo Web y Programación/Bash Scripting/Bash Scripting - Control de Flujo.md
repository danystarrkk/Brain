---
title: "Bash Scripting - Control de Flujo"
tags:
  - lang/bash
  - concept/control-flow
  - concept/loops
  - level/basico
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---
# Bash Scripting - Control de Flujo

> **Objetivo**: Dominar condicionales, bucles y control de ejecución en Bash.

---

## 1. Condicionales

### 1.1 `if` / `elif` / `else` / `fi`

```bash
# Sintaxis básica
if [[ condición ]]; then
    # comandos
elif [[ otra_condición ]]; then
    # comandos
else
    # comandos
fi

# En una línea (para casos simples)
if [[ -f archivo ]]; then echo "Existe"; fi
```

### 1.2 Operadores de Prueba en `[[ ]]`

```bash
# Archivos
[[ -e archivo ]]      # Existe (archivo, dir, link, etc.)
[[ -f archivo ]]      # Archivo regular
[[ -d dir ]]          # Directorio
[[ -L enlace ]]       # Enlace simbólico
[[ -r archivo ]]      # Legible
[[ -w archivo ]]      # Escribible
[[ -x archivo ]]      # Ejecutable
[[ -s archivo ]]      # Tamaño > 0
[[ archivo1 -nt archivo2 ]]  # archivo1 más nuevo que archivo2
[[ archivo1 -ot archivo2 ]]  # archivo1 más antiguo que archivo2
[[ archivo1 -ef archivo2 ]]  # Mismo inode (hardlinks)

# Strings
[[ -z "$str" ]]       # Vacío (longitud 0)
[[ -n "$str" ]]       # No vacío
[[ "$a" == "$b" ]]    # Igualdad (glob pattern en rhs)
[[ "$a" != "$b" ]]    # Desigualdad
[[ "$a" < "$b" ]]     # Orden lexicográfico (ASCII)
[[ "$a" > "$b" ]]
[[ "$str" == patrón ]]    # Match glob (ej: "archivo_*.txt")
[[ "$str" != patrón ]]    # No match glob
[[ "$str" =~ regex ]]     # Match regex (Bash 3.2+)

# Números (también en (( )))
[[ $a -eq $b ]]       # Igual
[[ $a -ne $b ]]       # Distinto
[[ $a -lt $b ]]       # Menor
[[ $a -le $b ]]       # Menor o igual
[[ $a -gt $b ]]       # Mayor
[[ $a -ge $b ]]       # Mayor o igual

# Lógicos
[[ cond1 && cond2 ]]  # AND
[[ cond1 || cond2 ]]  # OR
[[ ! cond ]]          # NOT
```

### 1.3 `(( ))` - Aritmética y Comparación Numérica

```bash
# Más natural para números, sintaxis estilo C
(( a = 5 + 3 ))       # Asignación con aritmética
(( a += 2 ))          # Incremento
(( a++ ))             # Post-incremento
(( ++a ))             # Pre-incremento

# Comparación (retorna 0/1 como exit code)
(( a < b ))           # true si a < b
(( a > b ))
(( a == b ))
(( a != b ))

# Uso en if
if (( contador > 10 )); then
    echo "Demasiados"
fi

# En bucles
for (( i=0; i<10; i++ )); do
    echo $i
done
```

### 1.4 `case` - Pattern Matching Múltiple

```bash
case "$variable" in
    patrón1)
        comandos
        ;;
    patrón2|patrón3)
        comandos  # OR con |
        ;;
    *[0-9]*)
        comandos  # Glob patterns
        ;;
    "")
        comandos  # String vacío
        ;;
    *)
        comandos  # Default (catch-all)
        ;;
esac

# Ejemplo práctico: menú
case "${1:-}" in
    start)   iniciar_servicio ;;
    stop)    detener_servicio ;;
    restart) reiniciar_servicio ;;
    status)  mostrar_estado ;;
    *)       echo "Uso: $0 {start|stop|restart|status}"; exit 1 ;;
esac
```

### 1.5 `select` - Menús Interactivos

```bash
PS3="Elige opción: "
opciones=("Opción 1" "Opción 2" "Salir")

select opt in "${opciones[@]}"; do
    case $REPLY in
        1) echo "Ejecutando opción 1" ;;
        2) echo "Ejecutando opción 2" ;;
        3) echo "Saliendo..."; break ;;
        *) echo "Opción inválida" ;;
    esac
done
```

---

## 2. Bucles

### 2.1 `for` - Iteración sobre Lista

```bash
# Sobre array (forma recomendada)
for elemento in "${array[@]}"; do
    echo "$elemento"
done

# Sobre rango numérico (Bash 3.0+)
for i in {1..10}; do
    echo $i
done

# Rango con paso (Bash 4.0+)
for i in {0..10..2}; do  # 0 2 4 6 8 10
    echo $i
done

# Estilo C (ver (( )))
for (( i=0; i<10; i++ )); do
    echo $i
done

# Sobre salida de comando (CUIDADO: word splitting)
for archivo in *.txt; do
    [[ -f "$archivo" ]] || continue  # Maneja caso sin matches
    echo "Procesando: $archivo"
done

# Iterar líneas de archivo (ROBUSTO)
while IFS= read -r linea; do
    echo "Línea: $linea"
done < archivo.txt
```

### 2.2 `while` - Condición de Entrada

```bash
# Contador
contador=0
while (( contador < 5 )); do
    echo $contador
    ((contador++))
done

# Leer archivo línea por línea (ESTÁNDAR ROBUSTO)
while IFS= read -r linea; do
    # Procesar $linea
done < archivo.txt

# Leer salida de comando (subshell - variables no persisten)
comando | while IFS= read -r linea; do
    # $linea disponible aquí
done
# Variables modificadas aquí NO persisten fuera del pipe

# Solución: process substitution (mantiene scope)
while IFS= read -r linea; do
    # Procesar
done < <(comando)

# Leer múltiples variables por línea (CSV, etc.)
while IFS=: read -r usuario pass uid gid info home shell; do
    echo "User: $usuario, Home: $home"
done < /etc/passwd
```

### 2.3 `until` - Inverso de while

```bash
# Ejecuta HASTA que condición sea verdadera
until [[ -f archivo_listo ]]; do
    echo "Esperando..."
    sleep 2
done
echo "Archivo listo"
```

---

## 3. Control de Bucles

### 3.1 `break` y `continue`

```bash
for i in {1..10}; do
    if (( i == 3 )); then
        continue  # Saltar iteración 3
    fi
    if (( i == 8 )); then
        break     # Salir del bucle en 8
    fi
    echo $i
done
# Output: 1 2 4 5 6 7

# break/continue con nivel (bucles anidados)
for i in {1..3}; do
    for j in {1..3}; do
        if (( j == 2 )); then
            break 2  # Sale de AMBOS bucles
        fi
        echo "i=$i, j=$j"
    done
done
```

### 3.2 `break`/`continue` en `case` dentro de bucles

```bash
while true; do
    read -p "Comando: " cmd
    case $cmd in
        salir) break ;;
        saltar) continue ;;
        *) echo "Ejecutando: $cmd" ;;
    esac
done
```

---

## 4. Patrones Avanzados

### 4.1 Bucle Infinito Controlado

```bash
# Patrón estándar para daemons/servidores
while true; do
    # Trabajo principal
    procesar_cola
    
    # Sleep con trap para shutdown graceful
    sleep 5 &
    wait $!  # Espera interruptible por señales
done
```

### 4.2 Iterar con Índice y Valor

```bash
# Arrays
for i in "${!array[@]}"; do
    echo "[$i] = ${array[i]}"
done

# Con contador manual
i=0
for elem in "${array[@]}"; do
    echo "$i: $elem"
    ((i++))
done
```

### 4.3 Procesar Archivos con `find` (Robusto para nombres con espacios)

```bash
# NUL-delimited (seguro para cualquier nombre)
while IFS= read -r -d '' archivo; do
    echo "Procesando: $archivo"
done < <(find . -type f -name "*.txt" -print0)

# O con mapfile (Bash 4+)
mapfile -d '' archivos < <(find . -type f -name "*.txt" -print0)
for archivo in "${archivos[@]}"; do
    echo "$archivo"
done
```

---

## 5. Trucos y Gotchas

| Situación | Problema | Solución |
|-----------|----------|----------|
| `for f in *.txt` | Falla si no hay matches (literal `*.txt`) | `shopt -s nullglob` antes o `[[ -f $f ]] \|\| continue` |
| `while read` en pipe | Variables no persisten | Usar `< <(comando)` process substitution |
| `read` consume backslashes | `\n` → `n` | Usar `read -r` siempre |
| `IFS` global afecta parsing | Cambia word splitting | `local IFS=...` o `IFS= read -r` |
| Bucles lentos en archivos grandes | Bash es lento línea a línea | Usar `awk`, `sed`, `mapfile` o `readarray` |

---

## 6. Referencias Cruzadas

- [Fundamentos Básicos](./Bash%20Scripting%20-%20Fundamentos%20B%C3%A1sicos.md) - Variables, arrays, plantilla base
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md) - Definición, parámetros, variables locales
- [Argumentos y Opciones](./Bash%20Scripting%20-%20Argumentos%20y%20Opciones.md) - getopts, parsing CLI
- [Procesos y Señales](./Bash%20Scripting%20-%20Procesos%20y%20Se%C3%B1ales.md) - Background jobs, trap, signals

---

*Etiquetas: #bash #control-flow #if #case #for #while #until #break #continue #select*