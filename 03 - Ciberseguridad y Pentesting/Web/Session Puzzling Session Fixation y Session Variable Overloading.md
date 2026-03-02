---
aliases:
  - Session Puzzling
  - Session Fixation
  - Session Variable Overloading
tags:
  - hacking/web
estado: 🟢 Terminado
---
# Gestión de Sesiones: Session Puzzling, Session Fixation y Session Variable Overloading

Session Puzzling, Session Fixation y Session Variable Overloading son diferentes nombres correspondientes a vulnerabilidades de seguridad que afectan la **gestión de sesiones** en una aplicación web.

## Session Fixation

La vulnerabilidad de **Session Fixation** se produce cuando un atacante establece un identificador de sesión válido para un usuario y luego espera a que el usuario inicie sesión. Si el usuario inicia sesión con ese identificador de sesión, el atacante podría acceder a la sesión del usuario y realizar acciones maliciosas en su nombre. Para lograr esto, el atacante puede engañar al usuario para que haga clic en un enlace que incluye un identificador de sesión válido o explotar una debilidad en la aplicación web para establecer el identificador de sesión.

> [!info] Mecanismo de Ataque
> Un atacante puede fijar una sesión de varias maneras:
> 1.  **URL Parameter**: `http://example.com/?JSESSIONID=attacker_session_id`
> 2.  **Cookie Injection**: Usando JavaScript (si no hay HttpOnly) o manipulando cabeceras HTTP.
> 3.  **Hidden Form Field**: En un formulario POST.
> El objetivo es que la víctima autentique la sesión que el atacante ya conoce o controla.

> [!tip] Prevención de Session Fixation
> La medida más efectiva es **regenerar el ID de sesión** después de una autenticación exitosa. Esto asegura que, incluso si un atacante fijó un ID de sesión antes del login, la sesión post-autenticación tendrá un ID completamente nuevo y desconocido para el atacante.

## Session Puzzling

El término “**Session Puzzling**” se utiliza a veces para referirse a la misma vulnerabilidad, pero desde el punto de vista del atacante que intenta **adivinar** o **generar identificadores** de sesión válidos.

> [!warning] Debilidades en la Generación de IDs de Sesión
> Los IDs de sesión deben ser impredecibles y tener suficiente entropía. Las debilidades comunes incluyen:
> *   **IDs secuenciales**: `1, 2, 3...`
> *   **IDs basados en tiempo**: `timestamp`
> *   **IDs basados en información predecible**: `hash(username + IP)`
> *   **IDs con baja longitud o caracteres limitados**: Fácilmente adivinables por fuerza bruta.
> Un atacante podría usar técnicas de fuerza bruta o análisis estadístico para predecir IDs válidos si el algoritmo de generación es débil.

## Session Variable Overloading

Por último, el término “**Session Variable Overloading**” se refiere a un tipo específico de ataque de Session Fixation en el que el atacante envía una gran cantidad de datos a la aplicación web con el objetivo de sobrecargar las variables de sesión. Si la aplicación web no valida adecuadamente la cantidad de datos que se pueden almacenar en las variables de sesión, el atacante podría sobrecargarlas con datos maliciosos y causar problemas en el rendimiento de la aplicación.

> [!info] Impacto y Mitigación
> La sobrecarga de variables de sesión puede llevar a:
> *   **Denegación de Servicio (DoS)**: Consumo excesivo de memoria del servidor.
> *   **Corrupción de Datos**: Si los datos maliciosos sobrescriben información legítima.
> *   **Ejecución de Código Remoto (RCE)**: En casos extremos, si la aplicación procesa los datos de sesión de forma insegura (ej. deserialización insegura).
> La mitigación implica **validación estricta de entrada** para todos los datos que se almacenan en la sesión, incluyendo límites de tamaño y tipo de datos.

## Prevención General de Vulnerabilidades de Sesión

Para prevenir estas vulnerabilidades, es importante utilizar identificadores de sesión aleatorios y seguros, validar la autenticación y autorización del usuario antes de establecer una sesión y limitar la cantidad de datos que se pueden almacenar en las variables de sesión.

> [!tip] Medidas de Seguridad Adicionales
> *   **Atributos de Cookies Seguras**:
>     *   `Secure`: Asegura que la cookie solo se envíe a través de HTTPS.
>     *   `HttpOnly`: Previene el acceso a la cookie desde JavaScript, mitigando ataques XSS.
>     *   `SameSite`: Protege contra ataques CSRF.
> *   **Tiempo de Vida de la Sesión**: Establecer tiempos de vida cortos para las sesiones y forzar la reautenticación para acciones sensibles.
> *   **Invalidación de Sesiones**: Invalidar explícitamente las sesiones al cerrar la sesión o cuando se detecta actividad sospechosa.
> *   **Monitoreo y Registro**: Implementar un monitoreo robusto para detectar patrones de ataque relacionados con sesiones.

## Notas Prácticas

![[Pasted image 20260227173616.png]]

Vamos a acceder con `admin:admin`:

![[Pasted image 20260227173746.png]]

Como observamos, tenemos una cookie de sesión la cual parece ser un [[JSON Web Tokens Explotación y Bypass|JSON Web Tokens]]. Veamos qué es lo que se puede hacer:

![[Pasted image 20260227173957.png]]

Observamos que tenemos un panel para cambiar la contraseña, donde vamos a capturar la petición primero:

![[Pasted image 20260227174157.png]]

Vemos cómo se tramita y, al cargar, nos dice:

![[Pasted image 20260227174213.png]]

Nos tiene que llegar un correo para cambiar la contraseña, pero si nosotros en el Burp Suite capturamos la respuesta:

![[Pasted image 20260227174305.png]]

Estamos observando una cookie de sesión y cómo esta la incrusta. Podemos intentar movernos a algún apartado específico como `dashboard` y, si lo arrastra, podríamos acceder:

![[Pasted image 20260227174452.png]]