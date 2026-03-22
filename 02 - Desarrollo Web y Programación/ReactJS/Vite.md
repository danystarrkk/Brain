---
aliases:
  - Vite
tags:
  - programacion/react
estado: 🟢 Terminado
---
## Bundler

Un bundler y específicamente la acción de bundling que es prácticamente que todos los componentes que se encuentran en nuestro proyecto son minificadas a una o unas salidas, un ejemplo gráfico seria el siguiente:

![[Pasted image 20260321223709.png]]

Como podemos observar tenemos varios archivos y todos se empaquetan a archivos que se interpretan `transpilan` en el navegador.

Entonces Vite es el bundler que vamos a usar e implementar en nuestro proyectos y automatiza el proceso de bundler para nosotros dedicarnos únicamente y de forma exclusiva en el código.

## Como funciona Vite

Para implementar vite tenemos que tener instalado `node` en nuestro pc para instalarlo de forma rápida, sus características son:

- Instant Server Start: utililiza ESM.
- Lightning Fas: Los cambios y los estados en componentes son mantenidos.
- Rich Features: Soporta TypeScript y JSX ademas de CSS y mas.
- Optimzed Build: pre compila la configuración.

## Crear Proyectos con vite


Para crear un proyecto primero tendremos que instalar mediante `npm` vite de la siguiente manera:

```bash
npm create vite@latest
```

![[Pasted image 20260321224815.png]]

como vemos seleccionamos la tecnología que necesitamos y cuando termine se despliega el servidor pero bueno si ingresamos en la carpeta que se genero para este proyecto vamos a ver lo siguiente:

![[Pasted image 20260321224941.png]]

Como podemos observar tenemos la misma estructura que ya se había visto.

bueno ya con esto si iniciamos el servidor con `npm run dev` podremos ver la web y configuración inicial que nos da:

![[Pasted image 20260321225239.png]]

Bueno tenemos el  archivo `package.json`, este es aquel que nos puede permitir agregar dependencias de desarrollo las cuales pro ser de desarrollo no llegan a producción a comparación de las demás dependencias:

![[Pasted image 20260321225357.png]]

Como vemos esto no es mas que un archivo `JSON` y tenemos algunas de las configuraciones que ya vimos en nuestra terminal al crear el proyecto, muchas de las cosas podemos ir alterando con forme sea necesario.

En la carpeta `public` vamos a encontrar imágenes archivos o assets necesarios para la web.

En `src` va a estar nuestros componentes.

El archivo principal y desde el cual se invoca a todo es mediante el `main.jsx`

El archivo donde unificaremos a todos los componentes es `app.jsx`.

## Cómo funcionan los Archivos Principales

El archivo `main.jsx` es el archivo mas importante y el principal ya que este importa solo lo necesario que en realidad son las librerías de react y al `App.jsx`.

El archivo `App.jsx` es el que contendrá todo el código como router y paginas que nuestro proyecto tendrá y luego esto lo importamos a `main.jsx`.