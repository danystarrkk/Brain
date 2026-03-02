---
aliases: ["DES-Yaml Attack", "YAML Deserialization Vulnerability"]
tags: [hacking/explotacion, programacion/python]
estado: 🟢 Terminado
---
# Ataque de Deserialización YAML (DES-Yaml)

Un **Ataque de Deserialización Yaml** (**DES-Yaml**) es un tipo de vulnerabilidad que puede ocurrir en aplicaciones Python que usan **YAML** (**Yet Another Markup Language**) para serializar y deserializar objetos.

La vulnerabilidad se produce cuando un atacante es capaz de controlar la entrada YAML que se pasa a una función de deserialización en la aplicación. Si el código de la aplicación no valida adecuadamente la entrada YAML, puede permitir que un atacante inyecte código malicioso en el objeto deserializado.

> [!info] ¿Qué es la Deserialización?
> La deserialización es el proceso de reconstruir un objeto a partir de una secuencia de bytes (o una cadena de texto en el caso de YAML). Cuando una aplicación deserializa datos de una fuente no confiable sin validación adecuada, un atacante puede insertar objetos maliciosos que, al ser reconstruidos, ejecuten código arbitrario.

Una vez que el objeto ha sido deserializado, el código malicioso puede ser ejecutado en el contexto de la aplicación, lo que puede permitir al atacante tomar el control del sistema, acceder a datos sensibles, o incluso ejecutar código remoto.

Los atacantes pueden aprovecharse de las vulnerabilidades de DES-Yaml para realizar ataques de denegación de servicio (DoS), inyectar código malicioso, o incluso tomar el control completo del sistema.

> [!warning] Riesgos de la Deserialización Insegura
> Las vulnerabilidades de deserialización son extremadamente peligrosas porque permiten la ejecución remota de código (RCE) en muchos casos. Esto se debe a que el atacante puede controlar la lógica de la aplicación al manipular los objetos que se crean en memoria.

El impacto de un Ataque de Deserialización Yaml depende del tipo y la sensibilidad de los datos que se puedan obtener, pero puede ser muy grave. Por lo tanto, es importante que los desarrolladores de aplicaciones Python validen y filtren adecuadamente la entrada YAML que se pasa a las funciones de deserialización, y que utilicen técnicas de seguridad como la limitación de recursos para prevenir ataques DoS y la desactivación de la deserialización automática de objetos no confiables.

## Notas de Práctica

![[Pasted image 20260227190542.png]]

Tenemos la siguiente web donde vemos en realidad algo extraño en la URL, por lo que vamos a copiar esa cadena y llevarla a la consola:

![[Pasted image 20260227190720.png]]

Vemos que tenemos el mismo texto que está en la web, claro, este se encuentra en Base64.

> [!tip] Decodificación Base64
> La codificación Base64 es comúnmente utilizada para transmitir datos binarios a través de medios que solo manejan texto. En el contexto de ataques web, a menudo se usa para ofuscar parámetros o payloads, requiriendo decodificación antes de su análisis o manipulación. Puedes decodificarlo fácilmente con `echo "cadena_base64" | base64 -d`.

Intentemos pasarle otra cosa usando la sintaxis a ver si esto funciona:

```bash
echo "Hola Mundo" | base64
```

![[Pasted image 20260227190837.png]]

Le pasamos el contenido a la URL:

![[Pasted image 20260227190859.png]]

Como vemos, sí se inserta el texto como esperábamos.

Ahora podemos asumir que se está confiando en el usuario, algo que no se debe hacer nunca.

Nosotros en YAML podemos usar sentencias maliciosas, las cuales pueden ser interpretadas y ejecutadas en el proceso.

Primero, la forma fácil en la que podemos pasar datos de texto usual a YAML es mediante Python de la siguiente manera:

Primero, debemos tener una estructura de datos como la siguiente:
![[Pasted image 20260227191452.png]]

Luego, realizamos lo siguiente dentro del intérprete de Python o mediante un script:

```python
import yaml
data = {'name': 'John Doe', 'age': 30}
yaml_data = yaml.dump(data)
print(yaml_data)
```

![[Pasted image 20260227191920.png]]

Actualmente, es necesario el uso de `safe_load()`. Esto es así en las últimas versiones y no solo es necesario, sino obligatorio para evitar la vulnerabilidad que estamos por explotar. En la web, vamos a hacer lo siguiente:

> [!warning] `yaml.load()` vs `yaml.safe_load()`
> La función `yaml.load()` (sin `safe_`) es inherentemente insegura porque puede deserializar cualquier objeto Python, incluyendo clases arbitrarias que pueden ejecutar código. `yaml.safe_load()` está diseñada para cargar solo un subconjunto seguro de YAML, evitando la creación de objetos arbitrarios y mitigando las vulnerabilidades de ejecución de código. Siempre se debe usar `yaml.safe_load()` al procesar YAML de fuentes no confiables.

```python
import yaml
import base64

# Payload malicioso para ejecutar un comando
# !!python/object/apply:subprocess.check_output ['ls -la']
malicious_yaml = "!!python/object/apply:subprocess.check_output ['ls -la']"

# Codificar el payload en Base64
encoded_payload = base64.b64encode(malicious_yaml.encode()).decode()
print(encoded_payload)
```

![[Pasted image 20260227192242.png]]

Vemos qué sucede si decodificamos eso:

![[Pasted image 20260227192307.png]]

Como observamos, logramos una ejecución de comandos.
