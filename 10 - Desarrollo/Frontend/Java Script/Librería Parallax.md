---
aliases: ["Parallax", "parallax.js", "Efecto parallax"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Librería GSAP]]"
  - "[[Html]]"
  - "[[Archivos SVG]]"
fuente: "https://github.com/wagerfield/parallax"
---

# Librería Parallax

El **parallax** es un efecto en el que el fondo se mueve a una velocidad diferente al contenido. El resultado es una ligera sensación de **profundidad** (un sutil estilo 3D) que revela partes de la escena que con un plano fijo no se percibirían.

Para lograrlo se acomodan los elementos en varias **capas** que se mueven de forma independiente (en sentido vertical u horizontal), generalmente controladas por el desplazamiento del usuario por la página, aunque también pueden moverse de forma automática (por tiempo o mouse).

> [!info] ¿Cuándo usarlo?
> Es ideal cuando se busca **inmersión**: contar una historia, promocionar un juego o producto, o dar carácter a una landing. Con moderación enriquece; en exceso dificulta la lectura.

Implementarlo a mano es largo (hay que calcular offsets en cada evento de scroll/mousemove), por eso usamos **[parallax.js](https://github.com/wagerfield/parallax)**, que recrea el efecto de forma mucho más rápida y sencilla.

## Instalación

No siempre hace falta instalar nada: podemos incluirlo mediante un `<script>` en nuestro [[Html|HTML]]:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/parallax/3.1.0/parallax.min.js"></script>
```

O instalarlo por npm cuando trabajemos con un proyecto y bundler:

```bash
npm i -s parallax-js
```

## Uso

1. En el HTML creamos el contenedor de las capas y le asignamos un `id`, por ejemplo `id="scene"`:

```html
<div id="scene">
  <img data-depth="0.2" src="fondo.png" alt="" />
  <img data-depth="1.0" src="personaje.png" alt="" />
</div>
```

2. A cada elemento interno le agregamos el atributo `data-depth`, que controla la **profundidad** (y por tanto cuánto se moverá respecto al puntero/scroll): valores bajos (`0.1`–`0.3`) para fondos lejanos, valores altos (`1`+) para primeros planos.

3. Finalmente inicializamos la instancia apuntando al elemento contenedor:

```js
const scene = document.getElementById('scene');
const parallaxInstance = new Parallax(scene);
```

> [!warning] Error típico de sintaxis
> La línea correcta lleva el signo `=`: `const scene = document.getElementById('scene');`. Sin él, JS lanza `SyntaxError: Unexpected identifier`. También verifica que el `id` del HTML y el del `getElementById` coincidan **exactamente**.

> [!tip] Instancia por ID o por elemento
> `new Parallax()` acepta tanto el elemento (`document.getElementById('scene')`) como el selector como string (`'#scene'`). Al destruir una instancia (por ejemplo al desmontar un componente) usa `parallaxInstance.destroy()` para liberar los listeners de eventos.

## Relacionadas

- [[Librería GSAP]] — animaciones y scroll con GSAP/ScrollTrigger
- [[Html]] — estructura de la escena con capas
- [[Archivos SVG]] — buenas candidatas para las capas del parallax
