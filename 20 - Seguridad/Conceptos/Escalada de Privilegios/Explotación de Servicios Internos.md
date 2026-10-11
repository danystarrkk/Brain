---
aliases:
  - Servicios Internos
  - Explotación de Servicios
tags:
  - hacking/escalada
  - linux/seguridad
estado: 🟢 Terminado
---
# Explotación de Servicios Internos

## Introducción a los Servicios Internos y su Seguridad

Los **servicios internos** son componentes esenciales que operan en segundo plano dentro de un sistema operativo, encargándose de funciones críticas como la gestión de red, impresión, actualización de software y monitoreo del sistema, entre otros.

> [!info] Tipos Comunes de Servicios Internos
> En sistemas operativos basados en Linux, los servicios suelen gestionarse a través de `systemd` (unidades de servicio), `init.d` o `xinetd`. En Windows, se conocen como "Servicios de Windows" y se gestionan a través del Administrador de Servicios (`services.msc`) o comandos como `sc`. Estos servicios pueden ejecutarse con diferentes niveles de privilegio, desde usuarios de bajo nivel hasta `root` o `SYSTEM`.

No obstante, si estos servicios **no están configurados adecuadamente** y se encuentran activos, pueden representar una brechas de seguridad significativa. Los atacantes podrían explotar estos servicios para obtener acceso no autorizado al sistema y llevar a cabo actividades malintencionadas.

> [!warning] Riesgos de Configuración Inadecuada
> Las configuraciones inadecuadas pueden incluir:
> *   **Permisos débiles**: Archivos de configuración o ejecutables de servicio con permisos de escritura para usuarios de bajo privilegio.
> *   **Credenciales por defecto o débiles**: Servicios que utilizan contraseñas predeterminadas o fáciles de adivinar.
> *   **Vulnerabilidades sin parchear**: Software de servicio obsoleto con exploits conocidos.
> *   **Configuraciones de red expuestas**: Servicios que escuchan en interfaces públicas cuando deberían ser solo locales.
> *   **Rutas de búsqueda inseguras (PATH hijacking)**: Servicios que ejecutan binarios sin rutas absolutas, permitiendo la inyección de ejecutables maliciosos.

Un ejemplo concreto sería un servicio de red mal configurado con permisos elevados. Si un atacante logra identificarlo y encontrar una forma de aprovecharlo, podría utilizarlo para escalar privilegios y obtener acceso de administrador.

> [!tip] Detección Inicial de Servicios
> Para detectar servicios activos en un sistema, se pueden utilizar diversas herramientas y comandos:
> *   **Linux**: `systemctl list-units --type=service`, `ps aux | grep -i service`, `netstat -tulnp` (para servicios de red).
> *   **Windows**: `sc query state= all`, `Get-Service` (PowerShell), `netstat -ano` (para servicios de red).
> La identificación de puertos abiertos y los procesos asociados es un paso crucial en el reconocimiento de servicios explotables.

En esta clase, analizaremos un caso ilustrativo de cómo un atacante podría, en primer lugar, detectar un servicio activo en el sistema y, posteriormente, explotarlo para incrementar sus privilegios de usuario.