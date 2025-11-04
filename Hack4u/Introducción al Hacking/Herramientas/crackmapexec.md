-----
**CrackMapExec** (CME) es una herramienta de post-explotación que ayuda a automatizar la evaluación de seguridad en grandes redes de Active Directory. Es conocida como "la navaja suiza para pentesting de AD".

## Tabla de Opciones Principales de CrackMapExec

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda|`crackmapexec -h`|
|`-id`|Credenciales en formato [domain/]user:password|`crackmapexec -id user:pass`|
|`-u`|Usuario|`crackmapexec -u usuario`|
|`-p`|Password|`crackmapexec -p password`|
|`-d`|Dominio|`crackmapexec -d DOMINIO`|
|`-H`|Hash NTLM|`crackmapexec -H aad3b435b51404eeaad3b435b51404ee:NT_HASH`|
|`-k`|Autenticación Kerberos|`crackmapexec -k`|
|`--aesKey`|AES key para Kerberos|`crackmapexec --aesKey [AES key]`|

### Opciones de Objetivo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-t`|Threads (default: 100)|`crackmapexec -t 50`|
|`--timeout`|Timeout (default: 20s)|`crackmapexec --timeout 30`|
|`--jitter`|Jitter entre requests|`crackmapexec --jitter 2`|
|`--delay`|Delay entre requests|`crackmapexec --delay 1`|
|`--source-ip`|IP de origen|`crackmapexec --source-ip 192.168.1.10`|
|`--port`|Puerto específico|`crackmapexec --port 445`|

### Opciones de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-v`|Modo verbose|`crackmapexec -v`|
|`--verbose`|Más verbose|`crackmapexec --verbose`|
|`-o`|Output file|`crackmapexec -o resultados.txt`|
|`--log`|Log file|`crackmapexec --log cme.log`|
|`--json`|Output JSON|`crackmapexec --json output.json`|
|`--csv`|Output CSV|`crackmapexec --csv output.csv`|
