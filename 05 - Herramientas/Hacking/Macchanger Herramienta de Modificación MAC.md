---
aliases:
  - macchanger
  - cambiar mac
tags:
  - linux/comandos
estado: 🟢 Terminado
---
# Macchanger: Herramienta para Modificación de Dirección MAC

Esta es una herramienta de línea de comandos en sistemas Linux que permite cambiar la dirección MAC (Media Access Control) de una interfaz de red.

> [!info] ¿Qué es una Dirección MAC?
> Una dirección MAC es un identificador único de 48 bits (6 octetos) asignado a la interfaz de red de un dispositivo para comunicaciones en la capa de enlace de datos del modelo OSI. Se utiliza para identificar de forma única un dispositivo en una red local (LAN). Los primeros 24 bits (3 octetos) suelen identificar al fabricante (OUI - Organizationally Unique Identifier), mientras que los últimos 24 bits son un identificador único para el dispositivo. Cambiar la dirección MAC puede ser útil para la privacidad, para eludir filtros de MAC en redes o para simular otro dispositivo.

## Parámetros

| Parámetro                     | Descripción                                                                                         |
| ----------------------------- | --------------------------------------------------------------------------------------------------- |
| `-s <Interfaz>` o `--show`    | Muestra la dirección MAC actual de la interfaz de red especificada.                                 |
| `-r <Interfaz>` o `--random`  | Cambia la dirección MAC de la interfaz de red a una dirección MAC aleatoria.                        |
| `-e <Interfaz>` o `--end`     | Fuerza un cambio completo de la dirección MAC, generando una nueva MAC aleatoria. Este modo asegura que la nueva MAC no coincida con ninguna OUI conocida, aumentando el anonimato. |
| `-p <Interfaz>` o `--permanent` | Restaura la dirección MAC original de fábrica de la interfaz.                                       |
| `-m <nueva Mac>` o `--mac`    | Cambia la dirección MAC de la interfaz a una dirección MAC específica proporcionada como argumento. |
| `-l` o `--list`               | Muestra todas las interfaces de red disponibles en el sistema.                                       |
| `-q` o `--quiet`              | Ejecuta `macchanger` en modo silencioso, suprimiendo la salida informativa.                        |

> [!tip] Uso Práctico y Consideraciones
> Para cambiar la dirección MAC de una interfaz, generalmente se requiere deshabilitarla primero, cambiar la MAC y luego volver a habilitarla. Esto se hace con comandos como `sudo ip link set <interfaz> down`, `sudo macchanger -r <interfaz>`, `sudo ip link set <interfaz> up`.
>
> > [!warning] Permisos y Persistencia
> > `macchanger` requiere privilegios de superusuario (root) para operar. Los cambios realizados con `macchanger` suelen ser temporales y se revierten al reiniciar el sistema o la interfaz de red, a menos que se configuren scripts de inicio o reglas udev para hacerlos persistentes.

En resumen, `macchanger` es una herramienta potente para modificar la dirección MAC de una interfaz de red en sistemas Unix y Linux, ofreciendo opciones para cambiar aleatoriamente la dirección MAC, restaurar la original de fábrica o especificar una dirección MAC personalizada.