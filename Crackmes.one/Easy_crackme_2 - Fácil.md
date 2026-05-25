## Análisis:

```bash
file cm1eng
# Info:
ELF 32-bit LSB executable
```

Uso objdump:

```bash
objdump -s -j .data ./cm1eng
```

Encuentro una string interesante:

```bash
QTBXCTU
```

Antes desensamblo el binario, enceuntro .text y lo dumpeo

```bash
objdump -d ./cm1eng
```

Info:

```nasm
Disassembly of section .text:

08048080 <.text>:
 8048080:       b8 04 00 00 00          mov    eax,0x4
 8048085:       bb 01 00 00 00          mov    ebx,0x1
 804808a:       b9 f8 90 04 08          mov    ecx,0x80490f8
 804808f:       ba 0d 00 00 00          mov    edx,0xd
 8048094:       cd 80                   int    0x80                 ;show Password: max. 0x0d (13) bytes
 8048096:       ba 00 01 00 00          mov    edx,0x100
 804809b:       b9 1b 91 04 08          mov    ecx,0x804911b
 80480a0:       bb 00 00 00 00          mov    ebx,0x0
 80480a5:       b8 03 00 00 00          mov    eax,0x3
 80480aa:       cd 80                   int    0x80                 ;wait for input

 ;the input is stored in 0x804911b

 80480ac:       be 26 91 04 08          mov    esi,0x8049126        ;offset of our magic word QTBXCTU
 80480b1:       89 f7                   mov    edi,esi              ;put address in edi
 80480b3:       31 db                   xor    ebx,ebx
 80480b5:       fc                      cld
 80480b6:       ac                      lods   al,BYTE PTR ds:[esi]     ;load byte of magic word
 80480b7:       34 21                   xor    al,0x21                  ;xor with 0x21
 80480b9:       aa                      stos   BYTE PTR es:[edi],al     ;store back
 80480ba:       43                      inc    ebx
 80480bb:       81 fb 07 00 00 00       cmp    ebx,0x7                  ;repeat 7 more times
 80480c1:       74 02                   je     0x80480c5
 80480c3:       e2 f1                   loop   0x80480b6
 80480c5:       be 1b 91 04 08          mov    esi,0x804911b            ;compare input from user with the password calculated
 80480ca:       bf 26 91 04 08          mov    edi,0x8049126
 80480cf:       b9 07 00 00 00          mov    ecx,0x7
 80480d4:       fc                      cld
 80480d5:       f3 a6                   repz cmps BYTE PTR ds:[esi],BYTE PTR es:[edi]
 80480d7:       75 16                   jne    0x80480ef                ;if not equal just bail out
 80480d9:       b8 04 00 00 00          mov    eax,0x4                  ;if equal print message
 80480de:       bb 01 00 00 00          mov    ebx,0x1
 80480e3:       b9 05 91 04 08          mov    ecx,0x8049105
 80480e8:       ba 16 00 00 00          mov    edx,0x16
 80480ed:       cd 80                   int    0x80
 80480ef:       b8 01 00 00 00          mov    eax,0x1                  ;syscall exit
 80480f4:       cd 80                   int    0x80

```

Funciones del programa:

- Escribe un mensaje en pantalla
- En base al input leido, el programa genera contraseña real desde “QTBXCTU”.
- Aplica XOR con 0x21
- Compara el string y lo valida con los requisitos

Información técnica:

Muestra mensaje `(0x08048094 int)`, Espera el input `(0x080480aa int)`, el input se guarda en (`0x804911b)`, Offset de QTBXCTU `(0x8049126)` para luego poner la dirección en `edi`.

Luego cada byte de la palabra correcta se carga desde `80480b6 lods` , Le hace un XOR con 0x21 en  `80480b7 xor` y lo devuelve a `80480b9 stos` para repetir ese proceso 7 veces en `80480bb cmp` . Ahora el input del usuario es comprado con el generado para ser la contraseña en  `80480c5 mov` 

Hay un break pero ahorita interesa solo eso.

Para encontrar la contraseña es necesario pasar los caracteres de “QTBXCTU” de ASCII (hex) a decimal:

```nasm
81, 84, 66, 88, 67, 84, 85 
```

Ahora, con los 7 caracteres (ejemplo con Q):

```nasm
Q = 81
0x21 = 33 
81 XOR 33 = 112
# En ASCII
"p"
```

Siendo P la primera letra de la contraseña.

Luego de repetir el proceso 6 veces más obtengo:

```nasm
pucybut
```

Listo.
