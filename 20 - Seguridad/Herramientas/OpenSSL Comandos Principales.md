---
aliases:
  - OpenSSL
tags:
  - herramientas/openssl
estado: 🟢 Terminado
---
# OpenSSL: Comandos Principales y Funciones Criptográficas

**OpenSSL** es un *conjunto de herramientas* de código abierto que implementa los protocolos SSL (Secure Sockets Layer) y TLS (Transport Layer Security), además de proporcionar una amplia gama de funciones criptográficas. Es la librería más utilizada para seguridad en internet, siendo fundamental para la comunicación segura en la web, VPNs, y muchas otras aplicaciones que requieren cifrado y autenticación.

> [!info] Importancia de OpenSSL
> OpenSSL no solo proporciona las herramientas de línea de comandos, sino que también es una biblioteca criptográfica subyacente utilizada por innumerables aplicaciones y servicios (como Apache, Nginx, Postfix, OpenSSH) para manejar operaciones SSL/TLS, cifrado de datos, generación de claves y gestión de certificados. Su robustez y flexibilidad lo convierten en un pilar de la ciberseguridad moderna.

## Tabla de Comandos Principales de OpenSSL

### Comandos Básicos

| Comando | Descripción | Ejemplo |
|---|---|---|
| `version` | Muestra la versión de OpenSSL instalada, incluyendo detalles sobre la configuración y las bibliotecas utilizadas. | `openssl version` |
| `help` | Proporciona una ayuda general sobre el uso de OpenSSL y lista los subcomandos disponibles. | `openssl help` |
| `list` | Lista todos los comandos y subcomandos disponibles en la instalación de OpenSSL. | `openssl list` |
| `-help` | Muestra la ayuda específica para un subcomando particular, detallando sus opciones y argumentos. | `openssl genrsa -help` |

> [!tip] Acceso a la Documentación
> Para obtener la documentación más detallada sobre cualquier subcomando de OpenSSL, se recomienda usar `man openssl` seguido del nombre del subcomando (ej. `man openssl genrsa`) en sistemas basados en Unix/Linux. Esto proporciona información exhaustiva sobre cada opción y su uso.

### Generación de Claves y Certificados

| Comando   | Descripción                                                                                                                                                                                                                  | Ejemplo                                                                                                            |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `genrsa`  | Genera una clave privada RSA. El tamaño de la clave (en bits) es crucial para la seguridad; 2048 o 4096 bits son los tamaños recomendados actualmente.                                                                       | `openssl genrsa -out key.pem 2048`                                                                                 |
| `genpkey` | Genera una clave privada de propósito general, soportando varios algoritmos como RSA, DSA, EC (Elliptic Curve). Es una alternativa más moderna a `genrsa`.                                                                   | `openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048`                                |
| `req`     | Se utiliza para generar una Solicitud de Firma de Certificado (CSR - Certificate Signing Request) o para crear certificados autofirmados. Una CSR es enviada a una Autoridad Certificadora (CA) para obtener un certificado. | `openssl req -new -key key.pem -out csr.pem -subj "/C=ES/ST=Madrid/L=Madrid/O=MiEmpresa/OU=IT/CN=www.example.com"` |
| `x509`    | Es una utilidad para gestionar, mostrar y convertir certificados X.509. Permite ver el contenido de un certificado, extraer información o cambiar su formato.                                                                | `openssl x509 -in cert.pem -text -noout`                                                                           |
| `ca`      | Actúa como una Autoridad Certificadora (CA) rudimentaria. Se utiliza para firmar CSRs y emitir certificados en un entorno de laboratorio o para una CA privada.                                                              | `openssl ca -in csr.pem -out cert.pem -config /etc/ssl/openssl.cnf`                                                |

> [!info] Componentes de un Certificado X.509
> Un certificado X.509 contiene información clave como la clave pública del titular, la identidad del titular (Subject), la identidad de la Autoridad Certificadora (Issuer), el período de validez, y la firma digital de la CA. La clave privada asociada a la clave pública del certificado debe mantenerse en secreto.

### Operaciones Criptográficas

| Comando | Descripción | Ejemplo |
|---|---|---|
| `enc` | Realiza cifrado y descifrado de archivos utilizando una variedad de algoritmos simétricos (ej. AES, DES, Triple DES). Permite especificar el modo de operación (CBC, GCM, etc.) y la clave/contraseña. | `openssl enc -aes-256-cbc -salt -in file.txt -out file.enc -pass pass:MiContraseñaSecreta` |
| `dgst` | Calcula resúmenes de mensajes (hashes) de archivos utilizando algoritmos como SHA256, SHA512, MD5. Es fundamental para verificar la integridad de los datos. | `openssl dgst -sha256 -hex file.txt` |
| `rsautl` | Realiza operaciones RSA utilizando una clave privada o pública. Se puede usar para cifrar, descifrar, firmar y verificar datos pequeños. | `openssl rsautl -encrypt -pubin -inkey public_key.pem -in plaintext.txt -out encrypted.bin` |
| `pkeyutl` | Una utilidad más general para operaciones con claves públicas y privadas, similar a `rsautl` pero compatible con más tipos de claves (RSA, DSA, EC). | `openssl pkeyutl -encrypt -pubin -inkey public_key.pem -in plaintext.txt -out encrypted.bin` |

> [!warning] Cifrado Simétrico vs. Asimétrico
> `enc` utiliza cifrado simétrico (misma clave para cifrar y descifrar), ideal para grandes volúmenes de datos. `rsautl` y `pkeyutl` utilizan cifrado asimétrico (par de claves pública/privada), más lento pero esencial para el intercambio seguro de claves y firmas digitales.

### Pruebas de Conectividad

| Comando | Descripción | Ejemplo |
|---|---|---|
| `s_client` | Actúa como un cliente SSL/TLS, permitiendo establecer una conexión segura a un servidor remoto. Es invaluable para depurar problemas de certificados, verificar la configuración TLS de un servidor y explorar los detalles de la conexión. | `openssl s_client -connect example.com:443 -showcerts -status` |
| `s_server` | Inicia un servidor SSL/TLS de prueba. Útil para probar clientes SSL/TLS, depurar configuraciones de certificados o simular un servicio seguro. | `openssl s_server -accept 4433 -cert cert.pem -key key.pem -www` |
| `s_time` | Mide el rendimiento de las operaciones SSL/TLS, como el handshake y la transferencia de datos, contra un servidor remoto. Ayuda a evaluar la latencia y el throughput. | `openssl s_time -connect example.com:443 -new` |

> [!tip] Depuración de Conexiones TLS
> El comando `openssl s_client` es una herramienta esencial para los pentesters y administradores de sistemas. Permite inspeccionar la cadena de certificados, los cifrados soportados por el servidor, la versión de TLS negociada y detectar posibles vulnerabilidades de configuración.

### Conversión de Formatos

| Comando | Descripción | Ejemplo |
|---|---|---|
| `pkcs12` | Maneja archivos en formato PKCS#12 (también conocidos como PFX o P12), que suelen contener una clave privada y su certificado correspondiente, a menudo protegidos por contraseña. Comúnmente usado para exportar/importar certificados en Windows o navegadores. | `openssl pkcs12 -export -out certificate.pfx -inkey key.pem -in cert.pem -name "Mi Certificado"` |
| `pkcs8` | Convierte claves privadas entre diferentes formatos, especialmente a y desde el formato PKCS#8, que es un estándar para el almacenamiento de claves privadas. | `openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in key.pem -out key_pkcs8.pem` |

> [!info] Estándares de Formato de Claves y Certificados
> Existen varios estándares para almacenar claves y certificados. PEM (Privacy-Enhanced Mail) es el formato más común en sistemas Unix/Linux, a menudo con extensiones `.pem`, `.crt`, `.key`. DER (Distinguished Encoding Rules) es un formato binario. PKCS#12 es un formato binario para almacenar una clave privada y su cadena de certificados. Comprender estos formatos es crucial para la interoperabilidad.