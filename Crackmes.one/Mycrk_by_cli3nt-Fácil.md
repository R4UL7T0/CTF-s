## Reconocimiento:

```bash
file mycrk
# Info:
ELF 32-bit LSB executable 
```

Lo ejecuto y pide una llave, así que de primeras veo si encuentro algo con strings.

```bash
strings mycrk
```

No encuentro nada interesante.

Pruebo con Ghidra para conocer en qué función esta y el nombre de la variable.

```bash
Función : main

Variable : local_c
```

Haciéndole click a la variable muestra el numero en decimal. Esa es la key:

```bash
5968496
```

 Listo.

Tip extra:

Puedes conocer el valor en decimal desde gdb:

```bash
(gdb) x/wd $ebp-0x8
# Info:
0xffffcd50: 5968496
```
