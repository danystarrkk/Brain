# Feature: Reorganización estructural del vault Brain

**Objetivo**: Scaffolding de la nueva estructura de carpetas (3 pilares) y migración física por pilar con `git mv`. Sin tocar contenidos de notas (frontmatter y tags quedan para fases posteriores, definidas por el usuario).

**Alcance autorizado (sesión 2026-02-11)**: Solo fases 1 y 2 del plan:
1. Scaffolding: carpetas + 3 plantillas en `40 - Plantillas/`
2. Migración por pilar: Desarrollo (02), Seguridad (03+05+06), Sistemas (04)

**Fuera de alcance**: frontmatter (fase 3), tags (fase 4), limpieza .trash/vacías/typos (fase 5), movimiento de Capturas 99→90.

## Tareas

- [x] 0. Baseline: branch `chore/vault-reestructuracion` + commit del estado previo no commiteado
- [x] 1. Scaffolding: carpetas raíz/subcarpetas + plantillas TPL-Concepto, TPL-Herramienta, TPL-Writeup
- [x] 2a. Migración pilar Desarrollo → `10 - Desarrollo/{Frontend,Backend,DevOps}`
- [x] 2b. Migración pilar Seguridad → `20 - Seguridad/{Conceptos,Herramientas,Writeups,Documentación}`
- [x] 2c. Migración pilar Sistemas → `30 - Sistemas/{Linux,Redes,Entorno}`

## Evidencia

- `d88a13e` chore(vault): sincronizar renombrados previos y estado actual del vault
- `7f75f2f` chore(vault): scaffolding de estructura 3 pilares y plantillas base
- `fd57dda` refactor(vault): migrar pilar Desarrollo (114 cambios, 100% renombres)
- `250f301` refactor(vault): migrar pilar Seguridad (121 cambios, 100% renombres)
- `9ef6b8b` refactor(vault): migrar pilar Sistemas (18 cambios, 100% renombres)

Totales: 10 - Desarrollo = 113 notas · 20 - Seguridad = 121 notas · 30 - Sistemas = 18 notas · 40 - Plantillas = 3 TPL + _TAGS_ESTRUCTURA.md. Working tree limpio.

## Pendiente (fases futuras, definidas por el usuario)

- Fase 3: frontmatter nota por nota
- Fase 4: unificación de tags
- Fase 5: limpieza (.trash → 99 - Archivo, carpetas vacías 08/Sin nombre, typos, Capturas 99 → 90)

## Notas

- Los `[[links]]` de Obsidian resuelven por nombre de archivo: mover carpetas no los rompe.
- Notas renombradas manualmente por el usuario (sin commitear): 18 borradas + 47 untracked; se sincronizan en el commit baseline.
