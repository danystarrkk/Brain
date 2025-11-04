---
tags:
  - FrontEnd
Creado: 2025-02-01
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Qué es una API?
Una API (Application Programming Interface) es un conjunto de reglas y protocolos que permiten la comunicación entre diferentes aplicaciones mediante código. Su propósito es facilitar la integración de servicios sin necesidad de crearlos desde cero.

Por ejemplo, si queremos integrar mapas en nuestra web, sería costoso y complejo desarrollar un servicio de mapas propio. En cambio, podemos utilizar la API de Google Maps, que nos permite acceder a funcionalidades ya desarrolladas a través de su SDK ( Software Development Kit es un conjunto de herramientas, librerías y documentación que facilita la integración de un servicio en una aplicación ) o endpoints( no es mas que el conjunto de reglas y especificaciones necesarios para la estructura de una petición receptada por la API para su correcta interpretación ).

En el caso de APIs web, su funcionamiento sigue el modelo Cliente-Servidor. Un Cliente (como una aplicación web o móvil) realiza peticiones a un Servidor a través de la API, respetando sus reglas y formatos. El Servidor procesa la solicitud y devuelve una respuesta estructurada, que el Cliente puede interpretar y utilizar según sea necesario.

## Qué es una API Rest
Una API REST (Representational State Transfer) es una API diseñada siguiendo la arquitectura REST. Esto significa que ha sido estructurada con un conjunto de reglas globales y estandarizadas, cuyo objetivo es asegurar la escalabilidad, facilidad de uso, seguridad y mantenibilidad. Por lo tanto, es importante conocer la estructura y las reglas de las APIs REST, ya que todas las que siguen esta arquitectura comparten principios similares, tanto en su desarrollo como en su consumo.

Las reglas principales de las API Rest son:
- **Uso de HTTP y método estándar**.
- **Recursos Accesibles mediante Urls**.
- **Representación de los recursos**.
- **Cacheabilidad**.
- **Interfaz Uniforme**.

## Funcionamiento de API Rest
Su funcionamiento es relativamente similar al como funciona una API donde la diferencia fundamental radica en la arquitectura que se basa la misma y las reglas que tenemos que seguir para sacarle el mejor partido a la misma, además la seguridad es un factor fundamental ya que se integraran cosas como las API keys que es la llave que nos permite usar la API y que se pueda identificar cual es el cliente que realiza ciertas peticiones, Otra cosa que se suele implementar las claves secretas o Authentication.

También tenemos que tener claro que se trabaja todo a nivel de servidor, esto quiere decir que no almacenaremos nada en el cliente, esta es una forma en la cual la información se puede mantener segura. Lo mas común en las respuestas es tener una respuesta en formato Json.

## Métodos
Cuando nosotros vamos a realizar una consulta como un cliente de una API Tenemos diferentes métodos los cuales son:
- **Get:** consultar datos. Nos permite obtener datos registrados en nuestra base de datos.
- **POST:** enviar datos. Nos permite insertar datos en nuestra base de datos.
-  **PUT**: Nos permite actualizar datos que se encuentren en la base de datos.
- **DELETE:** Nos permite eliminar datos de la base de datos.

## Códigos de Respuesta
Cuando realizamos consultas, estas nos darán un código conocido como "Código de Estado" mediante el cual identificaremos que paso con nuestra petición. Tenemos una gran variedad de códigos de estado los cuales podemos tener pero tenemos que comprender que no es necesario saberlos todos por lo que vamos aprenderos los generales:
- 2XX -> Estos códigos son usualmente EXISTOSO.
- 3XX -> Son de redireccionamiento, donde nos indica que la pagina cambio de url.
- 4XX -> Es usado cuando las consultas son invalidas o  erróneas.
- 5XX -> Están relacionados con fallos del servidor 