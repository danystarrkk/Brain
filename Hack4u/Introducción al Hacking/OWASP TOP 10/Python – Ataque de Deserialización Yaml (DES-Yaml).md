--------

Un **Ataque de Deserialización Yaml** (**DES-Yaml**) es un tipo de vulnerabilidad que puede ocurrir en aplicaciones Python que usan **YAML** (**Yet Another Markup Language**) para serializar y deserializar objetos.

La vulnerabilidad se produce cuando un atacante es capaz de controlar la entrada YAML que se pasa a una función de deserialización en la aplicación. Si el código de la aplicación no valida adecuadamente la entrada YAML, puede permitir que un atacante inyecte código malicioso en el objeto deserializado.

Una vez que el objeto ha sido deserializado, el código malicioso puede ser ejecutado en el contexto de la aplicación, lo que puede permitir al atacante tomar el control del sistema, acceder a datos sensibles, o incluso ejecutar código remoto.

Los atacantes pueden aprovecharse de las vulnerabilidades de DES-Yaml para realizar ataques de denegación de servicio (DoS), inyectar código malicioso, o incluso tomar el control completo del sistema.

El impacto de un Ataque de Deserialización Yaml depende del tipo y la sensibilidad de los datos que se puedan obtener, pero puede ser muy grave. Por lo tanto, es importante que los desarrolladores de aplicaciones Python validen y filtren adecuadamente la entrada YAML que se pasa a las funciones de deserialización, y que utilicen técnicas de seguridad como la limitación de recursos para prevenir ataques DoS y la desactivación de la deserialización automática de objetos no confiables.

## Notas de practica

![[Pasted image 20260227190542.png]]

Tenemos la siguiente web donde veos en realidad algo extraño en la URL por lo que vamos a copiar esa cadena y llevarla a la consola:

![[Pasted image 20260227190720.png]]

vemos que tenemos el mismo texto que esta en la web claro que este se encuentra en base64.

Intentemos pasarle otra cosa usando la sintaxis a ver si esto funciona:

![[Pasted image 20260227190837.png]]

le pasamos el contenido a la URL:

![[Pasted image 20260227190859.png]]

Como vemos si se inserta el texto como lo esperamos.

Ahora podemos asumir que se esta confiando en el usuario algo que no se debe hacer nunca.

Nosotros en YAML podemos usar sentencias maliciosas la cuales llegan a ser interpretadas y pues ejecutadas en el proceso.

Primero la forma fácil en la que podemos pasar datos de texto usual a YAML es mediante Python de la siguiente manera:

primero tenemos que tener una estructura de datos como la siguiente:
![[Pasted image 20260227191452.png]]

luego realizamos lo siguiente dentro del interprete de Python o mediante un script:

![[Pasted image 20260227191920.png]]

actualmente es necesario el uso de `safe_load()` esto en las ultimas versiones y no solo es necesario sino obligatorio para evitar la vulnerabilidad que estamos por explotar, donde la web vamos a hacer lo siguiente:

![[Pasted image 20260227192242.png]]

vemos que sucede si decodifica eso:

![[Pasted image 20260227192307.png]]

Como observamos pues logramos una ejecución de comandos.