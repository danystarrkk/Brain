---
aliases:
  - MageScan Opciones
  - MageScan Comandos
tags:
  - hacking/herramientas
  - hacking/web
estado: 🟢 Terminado
---
# Tabla de Opciones Principales de MageScan

> [!info] MageScan es una herramienta de código abierto diseñada para escanear sitios web basados en Magento en busca de vulnerabilidades, versiones, módulos instalados, parches faltantes y otra información relevante para la seguridad. Es fundamental para auditores de seguridad y desarrolladores que trabajan con plataformas Magento.

## Opciones Básicas

| Opción      | Descripción             | Ejemplo                               |
| ----------- | ----------------------- | ------------------------------------- |
| `-u`        | URL objetivo            | `magescan -u https://example.com`     |
| `--url`     | URL objetivo            | `magescan --url https://example.com`  |
| `-h`        | Mostrar ayuda           | `magescan -h`                         |
| `--help`    | Mostrar ayuda completa  | `magescan --help`                     |
| `-v`        | Modo verbose            | `magescan -v`                         |
| `--version` | Mostrar versión         | `magescan --version`                  |
| `--update`  | Actualizar base de datos| `magescan --update`                   |

> [!tip] Es una buena práctica ejecutar `magescan --update` regularmente para asegurar que la base de datos de vulnerabilidades y versiones esté al día, lo que mejora la precisión de los escaneos.

## Opciones de Escaneo

| Opción           | Descripción                 | Ejemplo                               |
| ---------------- | --------------------------- | ------------------------------------- |
| `scan:all`       | Escaneo completo            | `magescan scan:all`                   |
| `scan:version`   | Detectar versión            | `magescan scan:version`               |
| `scan:patches`   | Verificar parches           | `magescan scan:patches`               |
| `scan:modules`   | Enumerar módulos            | `magescan scan:modules`               |
| `scan:unreachable`| Buscar rutas inaccesibles   | `magescan scan:unreachable`           |
| `scan:sitemap`   | Analizar sitemap            | `magescan scan:sitemap`               |
| `scan:server`    | Información del servidor    | `magescan scan:server`                |

> [!info] El comando `scan:all` es un punto de partida excelente para una auditoría inicial, ya que combina varias verificaciones. Sin embargo, para análisis más profundos, es recomendable ejecutar los escaneos específicos (`scan:version`, `scan:modules`, etc.) de forma individual.

## Opciones de Salida

| Opción     | Descripción             | Ejemplo                               |
| ---------- | ----------------------- | ------------------------------------- |
| `-o`       | Archivo de salida       | `magescan -o resultados.txt`          |
| `--output` | Archivo de salida       | `magescan --output scan.txt`          |
| `--format` | Formato de salida       | `magescan --format json`              |
| `--json`   | Salida JSON             | `magescan --json output.json`         |
| `--xml`    | Salida XML              | `magescan --xml output.xml`           |

> [!tip] Utilizar los formatos de salida `--json` o `--xml` es ideal para integrar los resultados de MageScan con otras herramientas de análisis o para procesar los datos de forma programática.

## Opciones de Conexión

| Opción         | Descripción                 | Ejemplo                               |
| -------------- | --------------------------- | ------------------------------------- |
| `-t`           | Tiempo de espera de conexión| `magescan -t 30`                      |
| `--timeout`    | Tiempo de espera específico | `magescan --timeout 45`               |
| `--user-agent` | Agente de usuario personalizado| `magescan --user-agent "Mozilla/5.0"` |
| `--proxy`      | Usar proxy                  | `magescan --proxy http://proxy:8080`  |
| `--cookie`     | Cookie personalizada        | `magescan --cookie "admin=value"`     |

> [!warning] Al usar `--proxy`, asegúrate de que el proxy esté configurado correctamente y sea de confianza. Un proxy mal configurado o malicioso puede comprometer la privacidad y seguridad de tu escaneo. Además, el uso de `--cookie` puede ser útil para escanear áreas protegidas por autenticación, pero debe manejarse con precaución para evitar la exposición de credenciales.