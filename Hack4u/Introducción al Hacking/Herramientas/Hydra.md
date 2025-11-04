----

**THC-Hydra** es una herramienta de fuerza bruta paralelizada que admite numerosos protocolos de ataque. Es extremadamente rápida y flexible, permitiendo ataques de diccionario contra servicios de red.

## Tabla de Opciones Principales de Hydra

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-l`|Usuario específico|`hydra -l admin`|
|`-L`|Lista de usuarios|`hydra -L users.txt`|
|`-p`|Password específica|`hydra -p password123`|
|`-P`|Lista de passwords|`hydra -P pass.txt`|
|`-C`|File con user:password|`hydra -C creds.txt`|
|`-e`|Opciones adicionales|`hydra -e nsr`|
|`-t`|Tareas paralelas|`hydra -t 16`|
|`-w`|Timeout (segundos)|`hydra -w 10`|
|`-f`|Parar al encontrar creds|`hydra -f`|
|`-v`|Modo verbose|`hydra -v`|
|`-V`|Modo muy verbose|`hydra -V`|
|`-s`|Puerto específico|`hydra -s 8080`|
|`-S`|Usar SSL|`hydra -S`|
|`-4`|Usar IPv4 sólo|`hydra -4`|
|`-6`|Usar IPv6 sólo|`hydra -6`|

### Opciones de Output y Logging

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-o`|Output file|`hydra -o results.txt`|
|`-b`|Formato de output|`hydra -b json`|
|`-I`|Ignore restore file|`hydra -I`|
|`-O`|Usar formato de log antiguo|`hydra -O`|

### Opciones de Protocolos Específicos

| Opción | Descripción                     | Ejemplo                   |
| ------ | ------------------------------- | ------------------------- |
| `-m`   | Opciones específicas del módulo | `hydra -m "domain=LOCAL"` |
| `-M`   | File con múltiples objetivos    | `hydra -M targets.txt`    |
| `-u`   | Loop alrededor de usuarios      | `hydra -u`                |
| `-U`   | Información del módulo          | `hydra -U http-get`       |