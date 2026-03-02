---
aliases:
  - Tipos de IP y MAC
tags:
  - redes/conceptos
  - hacking/herramientas
  - linux/comandos
estado: 🟢 Terminado
---
Existen dos tipos de direcciones IP: una pública y una privada.

## Dirección IP Privada

Las direcciones IP privadas son asignadas localmente por nuestro router. Este tipo de dirección IP solo puede ser utilizada dentro de una red privada. No puede ser utilizada directamente por un atacante que no se encuentre en nuestro mismo segmento de red.

> [!info] Rangos de IP Privadas (RFC 1918)
> Las direcciones IP privadas están definidas por el RFC 1918 y no son enrutables en internet. Los rangos específicos son:
> *   **Clase A**: `10.0.0.0` a `10.255.255.255` (una única red /8)
> *   **Clase B**: `172.16.0.0` a `172.31.255.255` (16 redes /12)
> *   **Clase C**: `192.168.0.0` a `192.168.255.255` (256 redes /16)
> Estas direcciones permiten a las organizaciones tener sus propias redes internas sin agotar las direcciones IP públicas.

## Dirección IP Pública

Las direcciones IP públicas son únicas y se asignan a un dispositivo que tiene acceso a internet. Son globalmente visibles. Este tipo de dirección IP, al igual que la privada, sirve para la comunicación, pero a través de internet. Las direcciones IP públicas son asignadas por organizaciones regionales de Internet, conocidas como Registros Regionales de Internet (RIRs).

> [!tip] NAT y Direcciones IP Públicas
> La Traducción de Direcciones de Red (NAT) es un mecanismo crucial que permite a múltiples dispositivos en una red privada compartir una única dirección IP pública para acceder a internet. Cuando un dispositivo en la red privada envía tráfico a internet, el router reemplaza la dirección IP privada de origen con su propia dirección IP pública. Al recibir una respuesta, el router utiliza su tabla NAT para reenviar el paquete al dispositivo interno correcto.
>
> Las direcciones IP públicas pueden ser **dinámicas** (cambian periódicamente, asignadas por el ISP) o **estáticas** (fijas, a menudo usadas por servidores o servicios que requieren una dirección constante).

## Dirección MAC

Las direcciones MAC (Media Access Control) no son más que identificadores únicos asignados a los dispositivos de red. Esto es cierto, pero solo hasta cierto punto, ya que es posible cambiar o falsificar la dirección MAC de un dispositivo.

> [!info] Detalles Técnicos de la Dirección MAC
> Una dirección MAC es un identificador de 48 bits (6 bytes) grabado en el hardware de la interfaz de red (NIC) por el fabricante. Se utiliza en la capa de enlace de datos (Capa 2 del modelo OSI) para identificar de forma única un dispositivo dentro de un segmento de red local. Los primeros 24 bits (3 bytes) representan el Identificador Único de Organización (OUI) del fabricante, y los últimos 24 bits son un número de serie único asignado por el fabricante.
>
> > [!warning] Spoofing de MAC
> > La falsificación o "spoofing" de direcciones MAC implica cambiar la dirección MAC de una interfaz de red a una diferente de la que está grabada en el hardware. Esto se puede hacer por varias razones, como:
> > *   **Anonimato**: Ocultar la identidad real del dispositivo.
> > *   **Bypass de filtros**: Evadir filtros de seguridad basados en MAC en routers o puntos de acceso.
> > *   **Suplantación**: Hacerse pasar por otro dispositivo en la red.
> > *   **Acceso a redes**: Obtener acceso a redes que restringen el acceso a MACs específicas.
> > Es importante destacar que el spoofing de MAC solo afecta la comunicación dentro del segmento de red local y no la identificación a nivel de internet (que usa direcciones IP).

Para cambiar o falsificar direcciones MAC, se pueden usar herramientas como:
- [[Macchanger Herramienta de Modificación MAC|macchanger]]