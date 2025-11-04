----
Antes que nada las tareas cron son un tipo de tareas que se ejecutan en intervalos de tiempo. El saber leer este tipo de tares es algo complicado pero no imposible, vamos a aprender a como hacerlo, tomemos en cuenta que es algo parecido a [[Búsquedas y Expresiones Regulares]] al igual que los [[Re-directores y Descriptores de Archivos]].

La estructura para comprender las tareas cron los 5 asteriscos:
![[Pasted image 20240608221843.png]]

Como vemos los 5 asteriscos representan que la tarea se ejecuta cada segundo de cada minuto de cada hora por todos los dias del año.
Con esto claro tenemos que comprender que no solo pueden ir 5 asteriscos y que cada posición representa algo:
![[Pasted image 20240608221902.png]]

Dependiendo del valor que tengamos vamos a ver diferentes valores.
![[Pasted image 20240608221925.png]]

Como vemos se va a ejecutar cada 5 minutos en cada hora, es decir se ejecuta solo a la 1:05, 2:05, 3:05,…… y así asta terminar las 24 horas del día.
![[Pasted image 20240608221952.png]]
cada que veamos un asterisco y barra significa _cada_ por lo tanto se ejecuta siempre cada 5 minutos.