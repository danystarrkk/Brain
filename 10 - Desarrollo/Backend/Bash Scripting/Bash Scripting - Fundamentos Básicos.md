---
title: "Bash Scripting - Fundamentos Básicos"
tags:
  - lang/bash
  - concept/fundamentals
  - concept/syntax
  - level/basico
  - type/concept
  - area/scripting
created: 2024-01-15
updated: 2024-09-25
status: active
---

# Bash Scripting - Fundamentos Básicos

> **Objetivo**: Base sólida para escribir scripts Bash robustos, portables y mantenibles.

---

## 1. Shebang y Ejecución

```bash
#!/usr/bin/env bash
# #!/bin/bash  # Alternativa menos portable

# Hacer ejecutable
chmod +x script.sh
./script.sh
# o
bash script.sh
```

**Reglas de oro**:
- Siempre usar `#!/usr/bin/env bash` para portabilidad
- `set -euo pipefail` al inicio (ver [Debugging y Buenas Prácticas](./Bash%20Scripting%20-%20Debugging,%20Buenas%20Pr%C3%A1cticas%20y%20ShellCheck.md))

---

## 2. Variables y Tipos

### 2.1 Declaración Básica

```bash
# Convención: UPPER_SNAKE_CASE para variables de entorno/globales
# lower_snake_case para variables locales

nombre="Juan"
edad=30
pi=3.14159  # Bash solo maneja enteros nativamente

# Referenciar: SIEMPRE con comillas dobles
echo "Hola, $nombre"        # Expansión
echo 'Hola, $nombre'        # Literal (sin expansión)
```

### 2.2 `declare` / `typeset` - Tipado Explícito

```bash
declare -r READONLY_VAR="inmutable"      # Solo lectura (constante)
declare -i NUMERO=42                      # Entero (aritmética automática)
declare -a ARRAY_INDEXADO=(uno dos tres)  # Array indexado (0-based)
declare -A ARRAY_ASOCIATIVO=( [clave]=valor [otra]=otro )  # Array asociativo (Bash 4+)
declare -l LOWERCASE="MAYUSCULAS"         # Convertir a minúsculas al asignar
declare -u UPPERCASE="minusculas"         # Convertir a mayúsculas al asignar
declare -x EXPORT_VAR="exportada"         # Exportar a entorno (equivale a export)

# Ver atributos
declare -p NOMBRE_VAR
```

### 2.3 Arrays Indexados

```bash
# Declaración
frutas=(manzana banana cereza)
declare -a numeros=(1 2 3 4 5)

# Acceso
echo "${frutas[0]}"       # manzana (primer elemento)
echo "${frutas[-1]}"      # cereza (último, Bash 4.3+)
echo "${frutas[@]}"       # Todos los elementos (expansión segura)
echo "${#frutas[@]}"      # Longitud: 3

# Modificación
frutas[3]="durazno"       # Agregar en índice 3
frutas+=(uva pera)        # Append al final
unset 'frutas[1]'         # Eliminar elemento (deja hueco)
frutas=("${frutas[@]}")   # Reindexar (compactar huecos)

# Iteración segura
for fruta in "${frutas[@]}"; do
    echo "Fruta: $fruta"
done

for i in "${!frutas[@]}"; do
    echo "Índice $i: ${frutas[i]}"
done
```

### 2.4 Arrays Asociativos (Bash 4+)

```bash
declare -A colores=(
    [rojo]="#FF0000"
    [verde]="#00FF00"
    [azul]="#0000FF"
)

# Acceso
echo "${colores[rojo]}"       # #FF0000
echo "${colores[@]}"          # Todos los valores
echo "${!colores[@]}"         # Todas las claves: rojo verde azul
echo "${#colores[@]}"         # Cantidad: 3

# Iteración
for clave in "${!colores[@]}"; do
    echo "$clave -> ${colores[$clave]}"
done
```

---

## 3. Colores y Formato en Terminal

### 3.1 Códigos ANSI (Portable)

```bash
# Formato: \e[CÓDIGOm o \033[CÓDIGOm
reset="\e[0m"
bold="\e[1m"
dim="\e[2m"
underline="\e[4m"
blink="\e[5m"
inverse="\e[7m"
hidden="\e[8m"

# Colores foreground (30-37, 90-97)
black="\e[30m"; red="\e[31m"; green="\e[32m"; yellow="\e[33m"
blue="\e[34m"; magenta="\e[35m"; cyan="\e[36m"; white="\e[37m"
# Bright variants
b_red="\e[91m"; b_green="\e[92m"; b_yellow="\e[93m"; b_blue="\e[94m"

# Colores background (40-47, 100-107)
bg_black="\e[40m"; bg_red="\e[41m"; bg_green="\e[42m"; bg_yellow="\e[43m"
bg_blue="\e[44m"; bg_magenta="\e[45m"; bg_cyan="\e[46m"; bg_white="\e[47m"
```

### 3.2 Funciones Helper para Colores

```bash
# Uso: print_color "rojo" "Mensaje de error"
print_color() {
    local color="$1"; shift
    local color_code=""
    case "$color" in
        rojo|red) color_code="\e[31m" ;;
        verde|green) color_code="\e[32m" ;;
        amarillo|yellow) color_code="\e[33m" ;;
        azul|blue) color_code="\e[34m" ;;
        magenta) color_code="\e[35m" ;;
        cyan) color_code="\e[36m" ;;
        *) color_code="\e[0m" ;;
    esac
    printf "%b%s%b\n" "$color_code" "$*" "\e[0m"
}

# Ejemplo
print_color rojo "Error crítico"
print_color verde "Operación exitosa"
```

### 3.3 `tput` - Manipulación de Terminal (Más Portable)

```bash
# Colores
tput setaf 1   # rojo foreground
tput setaf 2   # verde
tput setab 4   # azul background
tput sgr0      # Reset all attributes

# Cursor
tput civis     # Ocultar cursor
tput cnorm     # Mostrar cursor (normal)
tput cup 10 20 # Mover cursor a fila 10, col 20
tput clear     # Limpiar pantalla
tput el        # Limpiar hasta fin de línea

# Dimensiones
cols=$(tput cols)
lines=$(tput lines)
```

---

## 4. Expansión de Parámetros (Parameter Expansion)

> **Clave para manipulación de strings sin herramientas externas**

```bash
var="archivo.tar.gz"

# Longitud
echo "${#var}"           # 14

# Substrings
echo "${var:0:7}"        # archivo (pos 0, len 7)
echo "${var: -3}"        # gz (últimos 3, espacio antes de - obligatorio)
echo "${var: -7:3}"      # tar (desde -7, len 3)

# Eliminar prefijo/sufijo (patrón glob)
echo "${var#*.}"         # tar.gz  (elimina prefijo más corto hasta .)
echo "${var##*.}"        # gz      (elimina prefijo más largo hasta .)
echo "${var%.*}"         # archivo.tar  (elimina sufijo más corto desde .)
echo "${var%%.*}"        # archivo      (elimina sufijo más largo desde .)

# Sustitución
echo "${var/tar/tgz}"    # archivo.tgz.gz  (primera coincidencia)
echo "${var//./_}"       # archivo_tar_gz  (todas las coincidencias)
echo "${var/#archivo/log}"  # log.tar.gz    (solo al inicio)
echo "${var/%.gz/.bz2}"   # archivo.tar.bz2 (solo al final)

# Valor por defecto
echo "${var:-default}"   # usa default si var vacía/no seteada
echo "${var:=default}"   # asigna default si vacía/no seteada
echo "${var:+alternativo}" # usa alternativo si var SÍ tiene valor
echo "${var:?error msg}"  # error si vacía/no seteada

# Mayúsculas/minúsculas (Bash 4+)
echo "${var^^}"          # ARCHIVO.TAR.GZ
echo "${var,}"           # archivo.tar.gz (solo primera)
echo "${var,,}"          # archivo.tar.gz (todas)
echo "${var^}"           # Archivo.tar.gz (primera letra)
```

---

## 5. Entrada de Usuario

```bash
# Básico
read -p "Nombre: " nombre

# Con timeout (segundos)
read -t 10 -p "Respuesta rápida: " resp || echo "Tiempo agotado"

# Ocultar entrada (passwords)
read -s -p "Contraseña: " pass; echo

# Límite de caracteres
read -n 1 -p "Continuar? [s/n]: " yn; echo

# Array desde entrada
read -a numeros -p "Números separados por espacio: "
echo "Total: ${#numeros[@]}"

# IFS personalizado para leer línea completa
IFS= read -r linea_completa
```

---

## 6. Buenas Prácticas Fundamentales

| Práctica | Por qué |
|----------|---------|
| `set -euo pipefail` | Falla rápido en errores, variables no seteadas, pipes rotos |
| Comillas dobles `"$var"` | Evita word splitting y glob expansion |
| `local` en funciones | Aísla variables, evita colisiones |
| `readonly` para constantes | Previene sobrescritura accidental |
| `declare -A` para mapas | Lookup O(1) vs O(n) en arrays indexados |
| `printf` sobre `echo` | Portable, control de formato, sin flags ambiguos |
| `[[ ]]` sobre `[ ]` | Más seguro, soporta `&&`, `||`, `==` glob, regex `=~` |
| `(( ))` para aritmética | Sintaxis C, no requiere `$` en variables |

---

## 7. Plantilla Base de Script

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# --- Configuración ---
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Colores (solo si stdout es terminal)
if [[ -t 1 ]]; then
    readonly RED=$'\e[31m' GREEN=$'\e[32m' YELLOW=$'\e[33m' BLUE=$'\e[34m' RESET=$'\e[0m'
else
    readonly RED="" GREEN="" YELLOW="" BLUE="" RESET=""
fi

# --- Funciones ---
log_info()  { printf "%b[INFO]%b %s\n" "$BLUE" "$RESET" "$*"; }
log_ok()    { printf "%b[OK]%b %s\n" "$GREEN" "$RESET" "$*"; }
log_warn()  { printf "%b[WARN]%b %s\n" "$YELLOW" "$RESET" "$*" >&2; }
log_error() { printf "%b[ERROR]%b %s\n" "$RED" "$RESET" "$*" >&2; }

cleanup() {
    # Limpieza al salir (trap EXIT)
    log_info "Limpiando recursos..."
}

trap cleanup EXIT

usage() {
    cat <<EOF
Uso: $SCRIPT_NAME [OPCIONES]
Opciones:
  -h, --help     Mostrar esta ayuda
  -v, --verbose  Modo verboso
EOF
}

main() {
    local verbose=false
    
    # Parseo de argumentos (ver nota dedicada)
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help) usage; exit 0 ;;
            -v|--verbose) verbose=true; shift ;;
            --) shift; break ;;
            -*) log_error "Opción desconocida: $1"; usage; exit 1 ;;
            *) break ;;
        esac
    done
    
    log_info "Iniciando $SCRIPT_NAME"
    # Lógica principal aquí
    log_ok "Completado"
}

main "$@"
```

---

## 8. Referencias Rápidas

| Construcción | Descripción |
|--------------|-------------|
| `$#` | Número de argumentos posicionales |
| `$@` | Todos los argumentos (array seguro) |
| `$*` | Todos los argumentos (string único, IFS) |
| `$?` | Exit code del último comando |
| `$$` | PID del shell actual |
| `$!` | PID del último proceso background |
| `$0` | Nombre del script |
| `$_` | Último argumento del comando anterior |

---

## 9. Ver También

- [Control de Flujo](./Bash%20Scripting%20-%20Control%20de%20Flujo.md) - if, case, bucles
- [Funciones y Scope](./Bash%20Scripting%20-%20Funciones%20y%20Scope.md)
- [Argumentos y Opciones](./Bash%20Scripting%20-%20Argumentos%20y%20Opciones.md)
- [Entrada-Salida y Redirección](./Bash%20Scripting%20-%20Entrada-Salida%20y%20Redirecci%C3%B3n.md)
- [Procesos y Señales](./Bash%20Scripting%20-%20Procesos%20y%20Se%C3%B1ales.md)
- [Debugging y Buenas Prácticas](./Bash%20Scripting%20-%20Debugging,%20Buenas%20Pr%C3%A1cticas%20y%20ShellCheck.md)

---

*Etiquetas: #bash #fundamentos #variables #arrays #colores #parameter-expansion #plantilla*