------------
**masscan** (Mass IP Port Scanner) es un escáner de puertos de altísima velocidad diseñado para escanear grandes rangos de IP en tiempos récord. Utiliza transmisión asíncrona y personalización de paquetes a nivel de kernel, permitiendo escanear toda Internet en minutos desde una sola máquina.

## Tabla de opciones principales de masscan

### Opciones Básicas de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-p`|Puertos a escanear|`masscan -p80,443 192.168.1.0/24`|
|`--ports`|Equivalente a `-p`|`masscan --ports 1-1000`|
|`--rate`|Paquetes por segundo a enviar|`masscan --rate 100000`|
|`-iL`|Lee objetivos desde archivo|`masscan -iL targets.txt`|
|`-oB`|Salida en formato binario|`masscan -oB output.bin`|
|`-oL`|Salida en formato lista|`masscan -oL output.txt`|
|`-oJ`|Salida en formato JSON|`masscan -oJ output.json`|
|`-oG`|Salida en formato grepable|`masscan -oG output.grep`|
|`-oX`|Salida en formato XML|`masscan -oX output.xml`|
|`--echo`|Muestra configuración actual|`masscan --echo`|

### Opciones de Configuración de Red

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--adapter`|Interfaz de red a usar|`masscan --adapter eth0`|
|`--adapter-ip`|IP específica del adapter|`masscan --adapter-ip 192.168.1.100`|
|`--adapter-port`|Puerto fuente a usar|`masscan --adapter-port 40000`|
|`--adapter-mac`|MAC address a usar|`masscan --adapter-mac 00:11:22:33:44:55`|
|`--router-ip`|IP del router/gateway|`masscan --router-ip 192.168.1.1`|
|`--router-mac`|MAC del router/gateway|`masscan --router-mac 00:11:22:33:44:55`|
|`--source-ip`|IP de origen personalizada|`masscan --source-ip 1.2.3.4`|

### Opciones de Rendimiento y Tuning

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--rate`|Tasa de paquetes por segundo|`masscan --rate 1000000`|
|`--wait`|Segundos a esperar al final|`masscan --wait 5`|
|`--connection-timeout`|Timeout de conexión|`masscan --connection-timeout 10`|
|`--retries`|Intentos de reenvío|`masscan --retries 2`|
|`--packet-trace`|Muestra paquetes enviados/recibidos|`masscan --packet-trace`|
|`--seed`|Semilla para generación aleatoria|`masscan --seed 12345`|
|`--regress`|Modo regresión (testing)|`masscan --regress`|

### Opciones de Escaneo Avanzado

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--banners`|Captura banners de servicios|`masscan --banners`|
|`--heartbleed`|Detecta vulnerabilidad Heartbleed|`masscan --heartbleed`|
|`--nmap`|Muestra comandos nmap equivalentes|`masscan --nmap`|
|`--ping`|Incluye descubrimiento de hosts|`masscan --ping`|
|`--exclude`|Excluye IPs/rangos|`masscan --exclude 192.168.1.100`|
|`--excludefile`|Excluye IPs desde archivo|`masscan --excludefile exclude.txt`|
|`--randomize-hosts`|Aleatoriza orden de IPs|`masscan --randomize-hosts`|
|`--randomize-ports`|Aleatoriza orden de puertos|`masscan --randomize-ports`|

### Opciones de Output y Formatos

| Opción             | Descripción                            | Ejemplo                        |
| ------------------ | -------------------------------------- | ------------------------------ |
| `-v`               | Verbose output                         | `masscan -v`                   |
| `--status-updates` | Intervalo de updates de estado         | `masscan --status-updates 1`   |
| `--iflist`         | Lista interfaces de red                | `masscan --iflist`             |
| `--resume`         | Resume escaneo desde archivo de estado | `masscan --resume paused.conf` |
| `--readscan`       | Lee archivo binario de escaneo         | `masscan --readscan scan.bin`  |