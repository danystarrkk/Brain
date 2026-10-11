# Feature: Reorganización estructural del vault Brain

**Objetivo**: Scaffolding de la nueva estructura de carpetas (3 pilares) y migración física por pilar con `git mv`. Sin tocar contenidos de notas (frontmatter y tags quedan para fases posteriores, definidas por el usuario).

**Alcance autorizado (sesión 2026-02-11)**: Solo fases 1 y 2 del plan:
1. Scaffolding: carpetas + 3 plantillas en `40 - Plantillas/`
2. Migración por pilar: Desarrollo (02), Seguridad (03+05+06), Sistemas (04)

**Fuera de alcance**: frontmatter (fase 3), tags (fase 4), limpieza .trash/vacías/typos (fase 5), movimiento de Capturas 99→90.

## Tareas

- [ ] 0. Baseline: branch feature + commit del estado previo no commiteado (renombrados manuales del usuario)
- [ ] 1. Scaffolding: carpetas raíz/subcarpetas + plantillas TPL-Concepto, TPL-Herramienta, TPL-Writeup — commit `chore(vault): scaffolding estructura y plantillas`
- [ ] 2a. Migración pilar Desarrollo → `10 - Desarrollo/{Frontend,Backend,DevOps}` — commit
- [ ] 2b. Migración pilar Seguridad → `20 - Seguridad/{Conceptos,Herramientas,Writeups,Documentación}` — commit
- [ ] 2c. Migración pilar Sistemas → `30 - Sistemas/{Linux,Redes,Entorno}` — commit

## Evidencia

(pendiente: identidades de commit al cerrar cada tarea)

## Notas

- Los `[[links]]` de Obsidian resuelven por nombre de archivo: mover carpetas no los rompe.
- Notas renombradas manualmente por el usuario (sin commitear): 18 borradas + 47 untracked; se sincronizan en el commit baseline.
