------
Esta es una herramienta en la terminal de sistemas Linux que nos permite cambiar la dirección MAC de una interfaz de red.

## Parámetros

| Parámetro                     | Descripción                                                                                         |
| ----------------------------- | --------------------------------------------------------------------------------------------------- |
| -s \<Interfaz\>  ó --show     | Nos permite Mostrar la dirección MAC actual de la interfaz de red especifica.                       |
| -r \<Interfaz\> ó --random    | Cambia la dirección MAC de la interfaz de red a una dirección Mac Aleatoria                         |
| -e \<Interfaz\> ó --end       | Forza un cambio completo de la dirección MAC, generando una nueva MAC aleatoria                     |
| -p \<Interfaz\> ó --permanent | Restaura la Dirección MAC original de la interfaz.                                                  |
| -m \<nueva Mac\> --mac        | Cambia la dirección MAC de la intrefaz a una dirección MAC específica proporcionada como argumento. |
| -l ó --list                   | Muestra todas las interfaces de red disponibles en el sistema.                                      |
| -q  ó --quiet                 | Ejecuta macchanger en modo silenciosos suprimiendo la salida informativa.                           |

En resumen, macchanger es una herramienta poderosa para modificar la dirección MAC de una interfaz de red en sistemas Unix y Linux, proporcionando opciones para cambiar aleatoriamente la dirección MAC, restaurar la original o especificar una dirección MAC personalizada.