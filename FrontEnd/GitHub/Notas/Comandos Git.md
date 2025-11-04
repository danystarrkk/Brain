---
tags: 
Creado: 2025-01-25
Relacionado:
  - "[[GITHUB]]"
---
----------
# Comandos de Git

|Comandos|Descripción|
|---|---|
|`git init`|Nos permite inicializar los repositorios.|
|`git status`|Nos brinda información sobre los commits, ramas y seguimiento de archivos.|
|`git add <Files>`|Nos permite dar seguimiento (staging) a uno o mas archivos.|
|`git reset <File>`|Nos permite sacar o eliminar un archivo del seguimiento.|
|`git commit -m "Mensaje"`|Nos permite hacer un commit del repositorio.|
|`git checkout -- .`|Nos permite reconstruir el proyecto a como estaba el ultimo commit.|
|`git branch`|Nos permite ver todas las ramas, incluida la rama actual.|
|`git branch -m master main`|Nos permite cambiar el nombre de una rama llamada master a main.|
|`git commit -am "Mensaje"`|Nos permite crear un commit siempre y cuando los archivos ya estén en seguimiento.|
|`git log`|Nos permite ver todos los commits que se han hecho.|
|`git diff`|Nos permite ver los cambios que no estén en el staged.|
|`git diff --staged`|Nos permite ver los cambios que están en el staged.|
|`git commit --amend -m "Mensaje"`|Nos permite actualizar el mensaje del ultimo commit echo.|
|`git reset --soft HEAD^`|Esto nos permite regresar un commit antes del ultimo commit.|
|`git reset --mixed <hash>`|Nos permite regresar a un commit anterior y dejar todos los cambios fuera del stage pero no eliminándolos de los archivos.|
|`git reset --hard <hash>`|Nos permite regresar a un commit anterior y eliminar todos los cambios que se tengan.|
|`git reflog`|Esto nos despliega una especie de historial el cual nos permite ver todas las modificaciones.|
|`git branch <nombre>`|Nos permite crear una nueva rama.|
|`git checkout <nombre>`|Nos permite movernos a una rama especifica.|
|`git merge <rama>`|Nos permite unir ramas, recordar estar en la rama a la que vamos a unir.|
|`git branch -d <nombre>`|Nos permite eliminar la rama que le pasemos.|
|`git checkout -b <nombre>`|Si no tenemos una rama con el nombre que definimos git nos la crea y nos mueve a esa rama.|
|`git tab <nombre del tag>`|Esto nos permite crear el tag en el ultimo commit.|
|`git tag -d <nombre del tag>`|Esto nos permite eliminar cualquier tag.|
|`git tag -a <version> -m <mensaje>`|Esto nos permite crear un tag de versiones y agregar un mensaje.|
|`git tag -a <version> <hash del commit> -m <Mensaje>`|Nos permite crear tags en commits específicos de nuestro repositorio.|
|`git tag`|Lista todos los tags.|
|`git show <nombre del tag>`|Nos permite visualizar un tag específico.|
|`git stash`|Esto nos permite crear el stash de las modificaciones que tenemos.|
|`git stash list`|Nos permite listar los stash.|
|`git stash pop`|Nos permite recuperar los cambios de nuestro stash.|
|`git stash clear`|Nos permite eliminar todos los stash sin preguntar.|
|`git stash apply <nombre del stash>`|En el caso de tener una cantidad grande de stash podemos usar lo que es|
|`git stash show <nombre del stash>`|Esto nos permite ver el archivo modificado y las líneas.|
|`git stash save <"Mensaje">`|Esto nos permite crear un stash mucho mas descriptivo.|
|`git stash list --stat`|Esto nos permite ver la información de todos los stash|
|`git stash drop <nombre del stash>`|Nos permite eliminar stash específicos.|
|`git rebase <nombre de la rama>`|Nos permite realizar un rebase de la rama actual hacia el puntero de la rama indicada.|
|`git rebase -i <hash del commit>`|Esto nos permite editar un commit de multiples forma como hacer un: squash, drop, edit, reword, etc.|
|||
|||
|||
|||
|||
|||