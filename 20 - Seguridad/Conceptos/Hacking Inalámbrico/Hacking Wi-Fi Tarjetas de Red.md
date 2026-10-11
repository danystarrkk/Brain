---
aliases:
  - Hacking Wi-Fi
  - Tarjetas de Red para Hacking Wi-Fi
tags:
  - hacking/inalambrico
  - hacking/herramientas
estado: 🟢 Terminado
---
# Hacking Wi-Fi: Requisitos de Hardware - Tarjetas de Red

Es fundamental comprender que para realizar hacking Wi-Fi, se puede utilizar cualquier tarjeta de red que admita el **modo monitor**. Este modo es crucial, ya que permite a la tarjeta capturar todos los paquetes de red en el aire, incluso aquellos que no están dirigidos específicamente a ella, lo cual es indispensable para técnicas como la inyección de paquetes y la captura de handshakes.

> [!info] Modo Monitor (Monitor Mode)
> El modo monitor es un modo de operación de una tarjeta de red inalámbrica que le permite escuchar todo el tráfico en un canal específico sin asociarse a un punto de acceso. A diferencia del modo promiscuo, que solo captura paquetes dirigidos a la interfaz o en la red a la que está conectada, el modo monitor opera a un nivel inferior, capturando tramas 802.11 crudas antes de que sean procesadas por el sistema operativo. Esto es vital para herramientas como Aircrack-ng, Kismet y Wireshark en el contexto de auditorías de seguridad inalámbrica.

Existen diversos tipos y marcas de tarjetas de red disponibles en el mercado. La clave no es la marca, sino la compatibilidad del chipset con el modo monitor y la inyección de paquetes.

> [!tip] Chipsets Recomendados
> Algunos de los chipsets más populares y compatibles para hacking Wi-Fi incluyen:
> *   **Atheros (AR9271, AR9300 series)**: Conocidos por su excelente soporte para modo monitor e inyección de paquetes.
> *   **Realtek (RTL8187L, RTL8812AU)**: También muy utilizados, especialmente el RTL8187L en tarjetas USB 802.11g y el RTL8812AU para 802.11ac.
> *   **Ralink (RT3070)**: Otro chipset con buen soporte.
> Al elegir una tarjeta, es recomendable verificar el chipset interno para asegurar la compatibilidad.

Una vez que se dispone de una tarjeta compatible, como las populares tarjetas **Alpha** (conocidas por su robustez, compatibilidad con el modo monitor y la inyección de paquetes, y su capacidad para operar con antenas externas de alta ganancia), se puede proceder con las herramientas y técnicas de auditoría inalámbrica.

> [!warning] Consideraciones Legales
> La realización de pruebas de seguridad en redes Wi-Fi sin autorización explícita es ilegal y puede tener graves consecuencias. Este material es estrictamente para fines educativos y de laboratorio controlado. Asegúrate siempre de tener permiso antes de realizar cualquier actividad de hacking en una red.