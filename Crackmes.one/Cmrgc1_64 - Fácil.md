## Análisis:

```bash
file 
# Info:
ELF 64-bit LSB executable
```

Este caso uso objdump:

```bash
objdump -d -Mintel ./cmrgc1_64
```

El programa inicia pidiendo la password:

```nasm
.text:08048824                     lea     edx, *input  ; password
.text:08048828                     mov     [esp+4], edx
.text:0804882C                     mov     [esp], eax
.text:0804882F                     call    ___isoc99_scanf
```

Luego hace el primer check ahí:

```nasm
.text:080488ED                     cmp     dword ptr [esp+2Ch], 39h
```

Donde esp+2c = input[1], continua y llega aquí:

```nasm
.text:08048794                     movzx   edx, byte ptr [esp+3Ah]
.text:08048799                     movzx   eax, byte ptr aT+4
.text:080487A0                     cmp     dl, al
.text:080487A2                     jnz     wrong
.text:080487A8                     movzx   edx, byte ptr [esp+3Bh]
.text:080487AD                     movzx   eax, byte_804B03D
.text:080487B4                     cmp     dl, al
.text:080487B6                     jnz     wrong
```

Donde compara input[2] == 0x27 y input[3] == 0x57, continua llegando aquí:

```nasm
.text:08048794                     movzx   edx, byte ptr [esp+3Ah]
.text:08048799                     movzx   eax, byte ptr aT+4
.text:080487A0                     cmp     dl, al
.text:080487A2                     jnz     wrong
.text:080487A8                     movzx   edx, byte ptr [esp+3Bh]
.text:080487AD                     movzx   eax, byte_804B03D
.text:080487B4                     cmp     dl, al
.text:080487B6                     jnz     wrong
```

Luego compara input [4] - 0x2e == 0x20. 

Continua y compara input[5] == 0x6d e input[7] == input [1].

Conociendo esto puedo generar el Keygen de esta manera:

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main() {
    int i;
    char pwd[8] = {0};
    srand(time(NULL));

    pwd[0] = (char)((rand()%(122-65))+65);
    pwd[7] = pwd[1] = pwd[0]^0x39;
    pwd[2] = 0x27;
    pwd[3] = 0x57;
    pwd[4] = 0x4e;
    pwd[5] = 0x6d;
    pwd[6] = 'r';
    for(i=0;i<8;i++) printf("\\x%.2x", pwd[i]);
    return 0;
}

```

Lo compilo:

```c
gcc keygen.c -o keygen
```

El output es:

```c
\x66\x5f\x27\x57\x4e\x6d\x72\x5f
```

Necesito convertirlo de Hexadecimal a ASCII:

```c
f_'WNmr_
```

Listo.
