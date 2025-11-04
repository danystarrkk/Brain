------
La vulnerabilidad **Remote File Inclusion** (**RFI**) es una vulnerabilidad de seguridad en la que un atacante puede **incluir** **archivos remotos** en una aplicación web vulnerable. Esto puede permitir al atacante ejecutar código malicioso en el servidor web y comprometer el sistema.

En un ataque de RFI, el atacante utiliza una entrada del usuario, como una URL o un campo de formulario, para incluir un archivo remoto en la solicitud. Si la aplicación web no valida adecuadamente estas entradas, procesará la solicitud y devolverá el contenido del archivo remoto al atacante.

Un atacante puede utilizar esta vulnerabilidad para incluir archivos remotos maliciosos que contienen código malicioso, como virus o troyanos, o para ejecutar comandos en el servidor vulnerable. En algunos casos, el atacante puede dirigir la solicitud hacia un recurso PHP alojado en un servidor de su propiedad, lo que le brinda un mayor grado de control en el ataque.

A continuación, se proporciona el enlace al proyecto de Github correspondiente al laboratorio que estaremos desplegando para practicar esta vulnerabilidad:

- **DVWP**: [https://github.com/vavkamil/dvwp](https://github.com/vavkamil/dvwp)

Asimismo, se os comparte el enlace directo para la descarga del plugin ‘**Gwolle Guestbook**‘ de WordPress:

- **Gwolle Guestbook**: [https://es.wordpress.org/plugins/gwolle-gb/](https://es.wordpress.org/plugins/gwolle-gb/)

## Notas de la practica

En este caso usamos una maquina la cual la modificamos un poco con el objetivo de que esta sea vulnerable a este tipo de ataque y en este caso lo primero que hicimos directamente es fuzzing para determinar posibles plugins en de WordPress en la web:

```bash
wfuzz -c --hc=404 -t 200 -w wp-plugins.fuzz.txt http://localhost:31337/FUZZ
```

![[Pasted image 20250825164045.png]]

como podemos ver tenemos un plugin certero y es el de `gwolle-gb`, para este plugin lo que vamos a hacer es uso de searchsploit para ver si existe alguna vulnerabilidad conocida:
![[Pasted image 20250825164203.png]]

como podemos observar si la tenemos y vamos a analizar que es lo que hacer exactamente pera explotarla:

![[Pasted image 20250825164448.png]]

la vulnerabilidad al parecer trata de que nosotros podemos mediante la url que nos brinda enviar peticiones a un servidor externo.

![[Pasted image 20250825164437.png]]

ahora lo que vamos a hacer es cargar un archivo wp-load.php el cual va a tener el siguiente contenido:

```php
<?php
	system('whoami');
?>
```

![[Pasted image 20250825164648.png]]

observamos que ya podemos ejecutar comandos.

de forma general es que la web nos interprete archivos externos con código malicioso.
