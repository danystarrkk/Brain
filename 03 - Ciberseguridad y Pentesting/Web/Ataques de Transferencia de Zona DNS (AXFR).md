---
aliases:
  - AXFR Attack
  - DNS Zone Transfer
tags:
  - hacking/reconocimiento
  - redes/conceptos
  - redes/protocolos
estado: 🟢 Terminado
---
# Ataques de Transferencia de Zona DNS (AXFR)

Los ataques de transferencia de zona, también conocidos como ataques **AXFR**, son un tipo de ataque que se dirige a los servidores **DNS** (**Domain Name System**) y que permite a los atacantes obtener información sensible sobre los dominios de una organización.

En términos simples, los servidores DNS se encargan de traducir los nombres de dominio legibles por humanos en direcciones IP utilizables por las máquinas. Los ataques AXFR permiten a los atacantes obtener información sobre los registros DNS almacenados en un servidor DNS.

> [!info] Fundamentos del DNS
> El Sistema de Nombres de Dominio (DNS) es un sistema de nomenclatura jerárquico y descentralizado para computadoras, servicios o cualquier recurso conectado a Internet o a una red privada. Su función principal es traducir nombres de dominio (ej. `google.com`) a direcciones IP (ej. `172.217.160.142`), lo que permite a los usuarios acceder a sitios web y otros recursos de red utilizando nombres fáciles de recordar en lugar de direcciones numéricas.

## ¿Cómo funciona un ataque AXFR?

El ataque AXFR se lleva a cabo enviando una solicitud de transferencia de zona desde un servidor DNS falso a un servidor DNS legítimo. Esta solicitud se realiza utilizando el protocolo de transferencia de zona DNS (AXFR), que es utilizado por los servidores DNS para transferir registros DNS de un servidor a otro.

> [!tip] Propósito legítimo de AXFR
> La transferencia de zona (AXFR) es una característica estándar del protocolo DNS diseñada para la replicación de bases de datos DNS entre servidores. Específicamente, permite que un servidor DNS secundario solicite una copia completa de la base de datos de una zona de un servidor DNS primario. Esto asegura la redundancia y la coherencia de los datos DNS en la red. El problema surge cuando esta funcionalidad no está restringida a servidores autorizados.

Si el servidor DNS legítimo no está configurado correctamente, puede responder a la solicitud de transferencia de zona y proporcionar al atacante información detallada sobre los registros DNS almacenados en el servidor. Esto incluye información como los nombres de dominio, direcciones IP, servidores de correo electrónico y otra información sensible que puede ser utilizada en futuros ataques.

> [!warning] Información sensible expuesta
> Una transferencia de zona exitosa puede revelar una gran cantidad de información valiosa para un atacante, incluyendo:
> *   **Subdominios**: Enumeración completa de subdominios (ej. `dev.ejemplo.com`, `mail.ejemplo.com`), lo que amplía la superficie de ataque.
> *   **Direcciones IP internas**: Si se utilizan registros DNS para recursos internos, estas IPs pueden ser expuestas, facilitando ataques de reconocimiento interno o pivoteo.
> *   **Servidores de correo (MX records)**: Identificación de servidores de correo, útiles para ataques de phishing o spoofing.
> *   **Servidores de nombres (NS records)**: Identificación de otros servidores DNS de la organización.
> *   **Registros TXT/SPF/DKIM**: Información sobre políticas de seguridad de correo electrónico o datos arbitrarios que pueden contener pistas.
> Esta información es crucial para la fase de reconocimiento y puede ser utilizada para planificar ataques más sofisticados, como ataques de fuerza bruta, ataques de denegación de servicio (DoS) dirigidos, o incluso para identificar vulnerabilidades en aplicaciones web o servicios específicos.

## Herramientas para la Explotación: `dig`

Una de las herramientas que utilizamos en esta clase para explotar el AXFR es **dig**. El comando `dig` es una herramienta de línea de comandos que se utiliza para realizar consultas DNS y obtener información sobre los registros DNS de un dominio específico.

> [!info] Alternativas a `dig`
> Además de `dig`, existen otras herramientas que pueden realizar consultas DNS y, en algunos casos, intentar transferencias de zona:
> *   **`host`**: Una utilidad más simple para realizar consultas DNS.
> *   **`nslookup`**: Otra herramienta de línea de comandos para consultar servidores DNS.
> *   **`fierce`**: Una herramienta de reconocimiento DNS escrita en Perl, diseñada para enumerar subdominios y, entre otras cosas, intentar transferencias de zona de forma automatizada.
> *   **`dnsenum`**: Una herramienta de enumeración DNS que puede realizar transferencias de zona, buscar registros, intentar fuerza bruta de subdominios, etc.

La sintaxis para aplicar el AXFR en un servidor DNS es la siguiente:

```bash
dig @<DNS-server> <domain-name> AXFR
```

Donde:

*   **`<DNS-server>`** es la dirección IP del servidor DNS que se desea consultar.
*   **`<domain-name>`** es el nombre de dominio del cual se desea obtener la transferencia de zona.
*   **`AXFR`** es el tipo de consulta que se desea realizar, que indica al servidor DNS que se desea una transferencia de zona completa.

## Prevención de Ataques AXFR

Para prevenir los ataques AXFR, es importante que los administradores de red configuren adecuadamente los servidores DNS y limiten el acceso a la transferencia de zona solo a servidores autorizados. También es importante mantener actualizado el software del servidor DNS y utilizar técnicas de cifrado y autenticación fuertes para proteger los datos que se transmiten a través de la red. Los administradores también pueden utilizar herramientas de monitoreo de DNS para detectar y prevenir posibles ataques de transferencia de zona.

> [!tip] Configuración de seguridad específica
> La medida de seguridad más crítica para prevenir ataques AXFR es configurar el servidor DNS para restringir las transferencias de zona. En servidores BIND, esto se logra mediante la directiva `allow-transfer` dentro de la configuración de la zona. Por ejemplo:
> ```
> zone "ejemplo.com" {
>     type master;
>     file "db.ejemplo.com";
>     allow-transfer { 192.168.1.10; 192.168.1.11; }; // Solo IPs de servidores secundarios autorizados
> };
> ```
> Si no hay servidores secundarios, se puede configurar `allow-transfer { none; };` o `allow-transfer { localhost; };` para denegar todas las transferencias externas. Es fundamental revisar periódicamente estas configuraciones y auditar los servidores DNS para asegurar que no haya zonas configuradas para permitir transferencias a cualquier origen (`allow-transfer { any; };`), lo cual es una vulnerabilidad grave.