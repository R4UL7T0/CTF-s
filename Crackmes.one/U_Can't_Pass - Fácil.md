Skills : Binary Patching

Es un binario que imprime dos mensajes, pero tiene uno oculto que gracias a una instrucción no imprime. El objetivo del ctf es anular (parchear) esa instrucción para que se impriman los 2 mensajes.

Primero abro el binario usando gdb:

```bash
gdb ./main
```

Dentro listo la función main:

```bash
(gdb) disas main
```

Info:

```nasm
0x0000555555555150 <+0>:	endbr64 
       0x0000555555555154 <+4>:	push   rbp
       0x0000555555555155 <+5>:	mov    rbp,rsp
    => 0x0000555555555158 <+8>:	sub    rsp,0x10
       0x000055555555515c <+12>:	mov    DWORD PTR [rbp-0x4],0x0
       0x0000555555555163 <+19>:	lea    rdi,[rip+0xe9a]        # 0x555555556004
       0x000055555555516a <+26>:	mov    al,0x0
       0x000055555555516c <+28>:	call   0x555555555050
       0x0000555555555171 <+33>:	mov    DWORD PTR [rbp-0x8],0xa
       0x0000555555555178 <+40>:	cmp    DWORD PTR [rbp-0x8],0xa
       0x000055555555517c <+44>:	je     0x55555555518e <main+62>
       0x000055555555517e <+46>:	lea    rdi,[rip+0xeab]        # 0x555555556030
       0x0000555555555185 <+53>:	mov    al,0x0
       0x0000555555555187 <+55>:	call   0x555555555050
       0x000055555555518c <+60>:	jmp    0x5555555551a4 <main+84>
       0x000055555555518e <+62>:	cmp    DWORD PTR [rbp-0x8],0xa
       0x0000555555555192 <+66>:	jne    0x5555555551a2 <main+82>
       0x0000555555555194 <+68>:	lea    rdi,[rip+0xe9f]        # 0x55555555603a
       0x000055555555519b <+75>:	mov    al,0x0
       0x000055555555519d <+77>:	call   0x555555555050
       0x00005555555551a2 <+82>:	jmp    0x5555555551a4 <main+84>
       0x00005555555551a4 <+84>:	mov    eax,DWORD PTR [rbp-0x4]
       0x00005555555551a7 <+87>:	add    rsp,0x10
       0x00005555555551ab <+91>:	pop    rbp
       0x00005555555551ac <+92>:	ret    
```

Vemos que se asigna `0xa` (10) a `[rbp-0x8]`. Luego se compara con `0xa` → siempre igual. La instrucción `je` (jump if equal) en `<+44>` salta **justo antes** de imprimir `"Success"`. Esto impide que el mensaje de éxito se muestre.

La solución aqui es cambiar je (jump if equal ) por jne ( jump if not equal) para que se salte la instrucción y se impriman los 2 mensajes.

Conociendo eso puedo comprobar que el espacio de memoria es el correcto para corregir:

```bash
(gdb) x/s 0x555555556030
# Info:
0x555555556030: "Success"
```

Listo, ahora si hago el patching.

Primero le doy la instrucción de que pare en el espacio de Memoria donde estaría je:

```bash
(gdb) break *0x000055555555517c
```

Ejecuto el programa y cuando pare sustituyo la instrucción:

```bash
(gdb) set {unsigned char}0x55555555517c = 0x75
```

Antes de seguir confirmo los cambios:

```bash
(gdb) x/i 0x55555555517c
# Info
0x55555555517c: jne    0x55555555518e
```

Ahora si continuo la ejecución y obtengo el resultado:

```bash
(gdb) continue
Continuing.
Success
[Inferior 1 (process 12345) exited normally]
```

Listo.

Mapa de solución:

```bash
INICIO
  |
  v
printf(mensaje_inicial)
  |
  v
[rbp-0x8] = 10
  |
  v
cmp 10, 10   -> ZF = 1 (iguales)
  |
  v
JNE (salta si diferentes)
  |
  |  ZF=1, entonces NO salta
  |
  v
printf("Success")  <- SI se ejecuta
  |
  v
EXITO
```
