---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Crea animaciones
Para la creación de animaciones tenemos que usar el conocido keyframes y para esto vamos a seguir 3 pasos:

```css
/*Definir la animacion y la duración*/
animation: animate 15s linear infinite;
animation-duration: calc(75s/var(--i));

/*Ya con la animación definida vamos definida usamos keyframes para definir como esta va a variar*/

/*Keyframe le damos el valor de la animación que seleccionamos y dentro del mismo va el punto inicial como si fuera una etiqueta y dentro el cambio que realizara*/

@keyframes animate {
  0% {
    /*realiza un transform donde se trasladara el 100 por ciento de la pantalla y la escala comenzara en 0*/
    transform: translateY(100vh) scale(0);
  }

  100% {
    /*en este punto le decimos que al llegar al 100% se traslade en y un poco mas, es por ello del valor negativo y su escala terminara máximo en 2*/
    transform: translateY(-10vh) scale(2);
  }
}
```