---
aliases: ["Secuestro de Librerías Python", "Python Library Hijacking"]
tags: ["hacking/explotacion"]
estado: 🟢 Terminado
---
# Python Library Hijacking

Cuando nos referimos a 'Python Library Hijacking', hablamos de una técnica de ataque que aprovecha la forma en que Python busca y carga bibliotecas para inyectar código malicioso en un script. El ataque se produce cuando un atacante crea o modifica una biblioteca en una ruta accesible por el script de Python, de tal manera que cuando el script la importa, se carga la versión maliciosa en lugar de la legítima.

## Mecanismo del Ataque

La forma en que el ataque se lleva a cabo es la siguiente: el atacante busca una biblioteca utilizada por el script y la reemplaza por su propia versión maliciosa. Esta biblioteca puede ser una biblioteca estándar de Python o una biblioteca externa descargada e instalada por el usuario. El atacante coloca su versión maliciosa de la biblioteca en una ruta accesible antes de que la biblioteca legítima sea encontrada.

> [!info] **Prioridad de Carga de Módulos**
> Python sigue un orden específico para buscar módulos y paquetes. Este orden es crucial para entender el "Library Hijacking":
> 1.  **Directorio actual**: El directorio desde donde se ejecuta el script.
> 2.  **PYTHONPATH**: Directorios especificados en la variable de entorno `PYTHONPATH`.
> 3.  **Rutas estándar de instalación**: Directorios donde Python instala sus bibliotecas estándar y de terceros (ej. `site-packages`).
> El atacante explota los puntos 1 y 2 para inyectar su código.

En general, Python comienza buscando estas bibliotecas en el directorio actual de trabajo y luego en las rutas definidas en la variable `sys.path`. Si el atacante tiene acceso de escritura en alguna de las rutas definidas en `sys.path`, puede colocar allí su propia versión maliciosa de la biblioteca y hacer que el script la cargue en lugar de la legítima.

Además, el atacante también puede crear su propia biblioteca en el directorio actual de trabajo, ya que Python comienza la búsqueda en este directorio por defecto. Si durante la carga de estas librerías desde el script legítimo, el atacante logra secuestrarlas, entonces conseguirá una ejecución alternativa del programa.

> [!warning] **Impacto y Mitigación**
> Un secuestro exitoso de una librería puede llevar a la ejecución de código arbitrario con los privilegios del usuario que ejecuta el script de Python. Para mitigar este riesgo:
> *   **Control de permisos**: Asegurar que los directorios en `sys.path` y el directorio de trabajo actual tengan permisos restrictivos.
> *   **Entornos virtuales**: Utilizar entornos virtuales (`venv`, `conda`) aísla las dependencias del proyecto, reduciendo la superficie de ataque.
> *   **Verificación de integridad**: Implementar mecanismos para verificar la integridad de las bibliotecas antes de su carga, aunque esto es más complejo de implementar en Python de forma nativa.
> *   **Minimizar `PYTHONPATH`**: Evitar añadir rutas no confiables a la variable de entorno `PYTHONPATH`.