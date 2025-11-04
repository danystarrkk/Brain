---
tags:
  - FrontEnd
Creado: 2024-01-25
Relacionado:
  - "[[GITHUB]]"
---
----------
# Configuración Inicial

Nosotros necesitamos configurar git como mínimo con un usuario y una contraseña de la siguiente forma:
```jsx
git config --global user.name "nombre de usuario"
git config --global user.email "danystarrkk@gmail.comm"
```

En el caso de necesitar cambiar la configuración, podemos modificar el archivo de git:
```jsx
git config --global -e
```

Otra configuración necesaria es cambiar el nombre de la rama principal a main:
```jsx
git config --global init.defaultBranch main
```