---
aliases: ["RWD", "Diseño responsive"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Medidas Relativas]]"
  - "[[Medidas CSS]]"
  - "[[Flexbox]]"
  - "[[CSS Grid]]"
fuente: "https://developer.mozilla.org/es/docs/Learn/CSS/CSS_layout/Responsive_Design"
---

# Responsive Web Design

**Responsive Web Design (RWD)** es la técnica para que una web se adapte a cualquier dispositivo: móviles, tablets, relojes, televisores, monitores. El punto de partida es una realidad: **son los usuarios quienes eligen con qué dispositivo navegan**; no podemos imponerles una resolución ni un navegador. Además, Google penaliza en posicionamiento a los sitios que no son responsive.

La solución se apoya en tres pilares:

1. **Vistas fluidas** — medidas relativas en vez de pixeles fijos (`%`, `rem`, `vw`) — ver [[Medidas CSS]] y [[Medidas Relativas]].
2. **Layouts flexibles** — `flex` y `grid` en vez de flotados rígidos — ver [[Flexbox]] y [[CSS Grid]].
3. **Media queries** — puntos de quiebre donde el CSS se ajusta.

## Media queries

Su sintaxis define una condición de pantalla que debe cumplirse para que las reglas apliquen:

![[Pasted image 20250124165031.png]]

Tenemos `media` y, entre paréntesis, la condición. En el ejemplo hay un `min-width`, que define un tamaño mínimo de pantalla: **las reglas se ejecutan cuando la pantalla tenga al menos 768px**.

```css
/* Estilos base (móvil primero) */
.columna {
  width: 100%;
}

/* Cuando la pantalla llegue a tablet */
@media (min-width: 768px) {
  .columna {
    width: 50%;
  }
}

/* Y en escritorio */
@media (min-width: 1140px) {
  .columna {
    width: 25%;
  }
}
```

### Breakpoints de referencia

No hay valores mágicos, pero estos son puntos de partida habituales:

| Px     | Dispositivo típico                          |
| ------ | ------------------------------------------- |
| 480px  | Teléfono                                    |
| 768px  | Tablet                                      |
| 1140px | Laptop y computadora de escritorio          |
| 1400px | Pantallas grandes                           |

> [!tip] Definí los breakpoints desde el contenido
> Los breakpoints salen de **dónde se rompe tu diseño**, no de un modelo de dispositivo específico. No existen "las medidas para iPhone y iPad": mirá tu contenido y elegí el punto donde deje de verse bien.

## Mobile-first: escribí primero el móvil

El enfoque recomendado es partir del diseño móvil (una columna, todo visible) e ir **añadiendo** complejidad con `min-width`. Lo opuesto (`max-width`) obliga a declarar el diseño completo y luego "quitar" — más código y más peleas con la cascada.

```css
/* ✅ Mobile-first: lo base + agregos */
@media (min-width: 768px) { ... }

/* ❌ Desktop-first: lo completo + restas */
@media (max-width: 767px) { ... }
```

## Buenas prácticas con media queries

- Reescribí **solo lo que cambia**: no dupliques selectores completos, solo la propiedad afectada.
- Colocá cada media query **justo debajo de la propiedad/ regla** que modifica, así se identifican de un vistazo.
- Usá `rem`/`%` dentro de las queries donde puedas: así respetan el tamaño de fuente que eligió el usuario.

## Más allá de las media queries: diseño fluido

Antes de escribir un breakpoint, preguntate si necesitas uno. Muchas cosas se resuelven fluidas, sin cortes:

```css
/* Un título que crece con la pantalla, con mín y máx — sin media queries */
h1 {
  font-size: clamp(2rem, 4vw + 1rem, 3.5rem);
}

/* Layouts que se reordenan solos */
.tarjetas {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 1rem;
}
```

Y las **container queries** llevan la idea al componente: un widget responsive **dentro de cualquier contenedor**, no solo de la ventana:

```css
.tarjeta {
  container-type: inline-size;
}

@container (min-width: 400px) {
  .tarjeta .contenido {
    display: grid;
    grid-template-columns: 120px 1fr;
  }
}
```

> [!info] No olvides el meta viewport
> Sin `<meta name="viewport" content="width=device-width, initial-scale=1">` en el HTML, los móviles simulan una pantalla de 980px y las media queries no hacen nada — ver [[Html]].

## Relacionadas

- [[Medidas Relativas]] — `vh`, `vw`, `%` y `clamp()` en detalle
- [[Medidas CSS]] — `px` vs `em` vs `rem` en un diseño adaptable
- [[Flexbox]] — layouts unidimensionales que fluyen
- [[CSS Grid]] — layouts bidimensionales con áreas responsivas
