-----

```bash
# Listar puertos abiertos
netstat -nat

ss -nltp

cat /proc/net/tcp

lsoft -i:<port>  #esto es para un puerto especifico
####################################################

#Listar porcesos del sistema

ps -faux

#####################################################

# Listar comandos ejecutados en el sistema

ps -eo command

##################################33333

# Conexiones con nc

nc -lnvp <puerto> #Nos permite dejar un puerto en escucha

nc <puert> #Nos permite conectarnos a un puerto

########################3

# Verificar comandos en tiempo real

watch -n 1 <comando> # nos permite ver cambios en tiempo real, lo suelo usar con ls -l

###################################

# Ver usuario conectados 

who

# Ver vaiables de entorno de mi sesion
printenv

# Ver la verción de kernel en el sistema
uname -r
```
