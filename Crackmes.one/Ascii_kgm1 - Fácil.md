## Reconocimiento:

```bash
file kgm1
# Info:
ELF 32-bit LSB executable 
```

```bash
gdb ./kgm1
```

El binario pide una key, por lo que estamos ante un keygen clásico.

Utilizo Ghidra para examinar sus funciones. Dentro encuentro la variable donde tiene que estar pero como tal no está, lo que encuentro es el como debe de estar.

Utilizo Objdump para encontrar mas datos:

```bash
objdump -d -Mintel ./kgm1
```

En el espacio 0x0804846E encuentro:

```nasm
cmp eax, 0xa
```

cmp compara dos valores.

eax normalmente contiene la longitud de algún input o buffer.

0xa es 10 en hexadecimal.

Sabiendo esto se revela que la longitud de la key es de 9 caracteres ya que en C/Assembly los strings suelen terminar con un “null termiantor ( **\0 )**”. Entonces si eax debe ser 10, eso incluye los 9 caracteres del string más 1 byte para el terminador **\0.**

Para cada caracter introducido el binario hace XOR con el valor en post[].

El valor del ultimo caracter tiene que ser igual a SUM(all xor result values).

Así podemos saltar 0x80484CF y que no arroje el mensaje “Invalid Key!”

Con esta info podemos crear el KeyGen.

Creo un script en Python:

```python
import random

def sign_extend(value, bits):
    sign_bit = 1 << (bits - 1)
    return (value & (sign_bit - 1)) - (value & sign_bit)

def is_key_valid(key):
    filt = 0x7ae311ccc8ab3645  # Definir dirección 0x8049707
    filt = filt.to_bytes(8, 'little')
  
    # bucle entre 0x0804847c y 0x0804848d 
    t = [ord(c) ^ s for c,s in zip(key, filt)]
    
    # bucle entre 0x0804849a y 0x080484a6 verifica la suma del caracter ya probado por XOR previamente 
    x = sum([sign_extend(c, 8) for c in t])
    
    # La suma resultante debe ser un carácter entre 'a' y 'z' y es el último carácter de la clave
    if x >= 97 and x <= 122:
        return True, key + chr(x)
    return False, None

def generate_key():
    string = ""
    for c in range(8):
        # Generar un caracter entre a - z
        s = random.randint(48, 122)  
        string += chr(s)
    return string

if __name__ == "__main__":
    # Generar clave mientras la clave no es válida
    key = generate_key()
    found, exact_key = is_key_valid(key) 
    while not found:
        key = generate_key()
        found, exact_key = is_key_valid(key)
    print("Exact key is : ", exact_key)

```

Generando así la key:

```python
4RKB:PS:f
```

Listo.
