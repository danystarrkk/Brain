--------
# Lectura de Permisos

Cuando nos referimos a la lectura de permisos tenemos nos referimos a identificar cuando podemos o no escribir, leer o ejecutar archivo o carpetas al igual que identificar el usuario propietario y el grupo al que pertenece, vamos a usar algunos [[Comandos, Rutas y Operadores]] y también [[Comandos Útiles]]

Para poder ver los permisos de un archivo vamos a hacer uso de:

```bash
ls -l
```

![[Pasted image 20240608214316.png]]

De donde nosotros vamos a identificar los permisos es de todo lo encerado en rojo.
Bueno en nuestro caso lo que tenemos es:
![[Pasted image 20240608214350.png]]
Primero vamos dividir esto en 3 pares:
![[Pasted image 20240608214417.png]]
Bueno ya con esto vemos que tenemos 3 pares de 3 y tenemos diferentes letra las cuales significan los siguiente:

- r → leer
- w → escribir
- x → ejecutar

Bueno tanto en el caso de directorios tanto w como x tenemos que tener cuidado, ya que si un directorio tiene w quiere decir que no podemos crear archivos dentro del mismo y en el caso de la x de ejecución es lo que nos permite ingresar al directorio, en el caso de archivos es bastante intuitivo.

Bueno Ahora centrémonos en nuestras tres secciones las cuales se dividen en:

- Propietario
- Grupo
- Otros

![[Pasted image 20240608214458.png]]

y bueno para ver tanto el propietario y el grupo lo vemos en la siguiente parte:
![[Pasted image 20240608214521.png]]
Ya con esto ya comprendemos y sabemos identificar el grupo y el usuario, una cosa a tener en cuenta es que no siempre el usuario es el mismo del grupo.
# Asignando permisos

Para nosotros cambiar los permisos de diferentes archivos vamos a usar el siguiente comando:

```bash
chmod <u,g,o> + <r,w,x> <archivo o carpeta>

# Ejemplo

chomd u+x file #Asignamos al usuario el permiso de escritura.
chmod o-w file #Quitamo el permiso de escritura para los grupos.

# Si queremos asignar varios permisos a laves podemos hacer:
chomod u-r,o+x,g+x #Con esto al usuario quitamos la lectura, a otros asignamos ejecución y a los grupos asignamos ejecucíon.

# Tambien en cada asignación hacer varios:
chomod u-rx,o+xw,g+xr
```

Ya con esto comprendemos como podemos modificar los los permisos del archivo de forma fácil, tenemos que saber que para modificar los permisos de usuario tenemos que se el propietario del archivo, para modificar los permisos de grupos pertenecer al grupo asignado y para modificar otros ser parte del grupo o ser el propietario del archivo.

## Asignando Permisos Octal

Esta es otra forma de asignación de permisos la cual vamos a aprender.

Vamos a esto a convertir a ceros y unos done uno tenemos un permisos y donde 0 no tenemos ninguno:
![[Pasted image 20240608214543.png]]
Ahora por debajo de cada uno vamos vamos a poner indices comenzando con el 0:
![[Pasted image 20240608214603.png]]
Por ultimo solo tomamos en cuenta lo que tiene un uno y vamos a elevar el 2 al índice que tenemos quedaría así:
![[Pasted image 20240608214620.png]]
Por ultimo sumamos los permisos:
![[Pasted image 20240608214639.png]]
Ya con esto claro hacemos los damas y debería quedarnos de la siguiente forma:
![[Pasted image 20240608214700.png]]
Esta es la forma larga, otra forma es asignarle un valor a cada permiso y luego sumarlo, los valores quedan de la siguiente manera:

- r → 4
- w → 2
- x → 1

Con estos valores solo debemos comprender que si el permiso esta asignado lo sumamos y si no no lo hacemos con esto es mucho mas fácil sacar el valor.

Bueno y para signar los permisos lo hacemos de la siguiente manera:

```bash
chmod 755 <nombre del archivo> #El 755 lo cambiamos por el numero para los permisos que queramos.
```

# **Permisos especiales – Sticky Bit**

Los permisos sticky bit es un tipo de permiso especial el cual se puede asignar a directorios o archivo, esto nos permite que los archivos dentro de un directorio no puedan ser ni borrados ni alterados por mas que los archivos dentro tengan permisos de escritura o ejecución.

Para asignar este tipo de permiso podemos hacerlo de la siguiente manera:

```bash
chmod +t <nombre del directiorio>
```

Otra forma de asignar esto es ubicando un 1 por delante de los demás permisos:

```bash
chmod 1755 <nombre del archvo> #No es nesesario cambiar los otros permisos del archivo podemos asignar los mismos pero con el 1 delante para que asigne el sticky bit
```

# **Control de atributos de ficheros en Linux – Chattr y Lsattr**

Bueno de manera general son atributos los cuales asignamos con Chattr y con lsattr los listamos.

La forma de asignar estos atributos lo podemos hacer de la siguiente forma:

```bash
chattr +i -V <nombre del archivo o carpeta> #Con esto asignamos el atributo de inmutable y esto es inmutable incluso para root, -V solo es para que nos liste información de lo que iso.

Tenemos varios como son:
+i -> inmutables.
+c -> no se comprime automaticamente.
+u -> cuando el fichero se borre se guarden sus datos.
+s -> cuando el archivo se borre se llenan los bloques uzados con ceros.
+S -> Los cambios se almacenan en el disco de forma sincronica y directa.

```

Para listar los atributos lo hacemos con de la siguiente manera:

```bash
lsattr <nombre del archivo o directorio>
```

---

# **Permisos especiales – SUID y SGID**

## Permisos SUID

los permisos SUID son un tipo de permiso especial el cual nos permite que los binarios o archivos se ejecuten de tal forma como si fusemos el propietario.

Para asignar este tipo de permisos lo hacemos de la siguiente forma:

```bash
chmod u+s <binario o archivo> #Con esto asignamos el permiso.

#Otra forma de asignarlo es de la siguiente manera:

chmod 4755 <binario o archivo> #Tomamos en cuenta que seasigna un 4 delante de los demas permisos.
```

## Permisos SGUID

Este tipo de permisos tiene dos funciones:

- Si se encuentra en un archivo permite que cualquier usuario que lo ejecute lo haga como si fuera miembro del grupo.
- Si esto se asigno en un directorio, cualquier archivo dentro del mismo se le asigna el mismo grupo que el del directorio.

Para asignar este tipo de permiso lo hacemos de la siguiente manera:

```bash
chmod g+s <binario o archivo> #Con esto asignamos el permiso

#Otra forma de asignarlo es de la siguiente manera:

chmod 2755 <binario o archivo> #Tomamos en cuenta que seasigna un 2 delante de los demas permisos.
```

# Privilegios especiales - Capabilities

Las Capabilities son ciertas capacidades que se le pueden asignar a un binario, estas nos permiten realizar cierto tipo de tareas de forma privilegiada.

Para poder listar este tipo de tareas podemos usar la herramienta de getcap.

Para asignar Capabilities podemos hacer:

```bash
setcap <capabilitie> <archivo>
```

Para listar, con getcap como se dijo.