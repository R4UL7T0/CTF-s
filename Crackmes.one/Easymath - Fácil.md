## Análisis:

```bash
file easymath
# Info:
ELF 32-bit LSB executable
```

Uso Ghidra para examinar el código:

```bash
ghidra
```

Dentro de la función “FUN_08048438” encuentro esto:

```c
else {
	iVar3 = atoi (*(char **)(param_2 +4));
	if (iVar3 * 0xc == 0x4530) {
		puts("done");
		}
	}
}
return 0;
}  
```

Ocupamos que salga el mensaje de “done” ya que al ingresar un input invalido no sale nada.

Inspeccionando el código se ve que el numero correcto está en la variable iVar3 que será representado por x:

```c
x * 12 = 17712
```

Con la siguiente operación matemática: 

17712 (0x4530) dividido entre 12 (0xc)

x / clave es 1476

Listo.
