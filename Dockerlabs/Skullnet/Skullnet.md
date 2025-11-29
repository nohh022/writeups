# 🖥️ Write-Up: [SKULLNET](https://dockerlabs.es)

## 📌 Información General
    - Nombre de la máquina: Skullnet
    - Plataforma: Dockerlabs
    - Dificultad: Difícil
    - Creador: Slayer0x & W4tson
    - OS: Linux
---

## 🔍 Enumeración

La máquina Skullnet posee la ip 172.18.0.2

### Descubrimiento de Puertos

Realizamos un reconocimiento de todos los puertos de la máquina y nos quedamos únicamente con aquellos que están abiertos. Para ello, vamos a recurrir a la herramienta **nmap**.

![allPorts](screenshots/allPorts.png)

Observamos que la máquina únicamente tiene abierto el puerto 80, así que vamos a analizar el servicio y versión que se ejecuta en él con la misma herramienta de **nmap**.

![target](screenshots/target.png)


### Puerto 80

El puerto 80 está ejecutando un servicio web con Apache. Además, vemos que se aplica **Virtual Hosting**, así que vamos a añadir la ip de la máquina y el dominio **skullnet.es** al archivo **/etc/hosts**

![etc-hosts](screenshots/etc-hosts.png)

Ahora ya podemos acceder a ese dominio desde nuestro navegador.

![home-80](screenshots/home-80.png)

Tras revisar el código fuente no encontramos nada interesante.

Vamos a proceder a la enumeración de subdirectorios utilizando la herramienta **gobuster**. Tras un rato probando diferentes diccionarios, al utilizar el **common.txt** del repositorio **SecLists** junto con la extensión **git**, encontramos una carpeta que parece ser un repositorio de Github.

![fuzzing-git](screenshots/fuzzing-git.png)

Al acceder a ella con el navegador confirmamos que se trata de un repositorio.

![git-subdirectory](screenshots/git-subdirectory.png)

Descargarnos el repositorio utilizando la herramienta **wget**.

`wget -r http://skullnet.es/.git/`

Vamos a utilizar `git log` para ver el historial de commits que se han ejecutado en ella.

![git-log](screenshots/git-log.png)

Podemos ver que se han realizado 2 commits, para poder ver su contenido utilizamos `git show` más el **ID** de cada commit.

Vemos que aparecen dos archivos que ya no están en el repositorio, **authenticator.txt** y **network.pcap**.

Recuperamos esos archivos utilizando:

```bash
git restore authentication.txt network.pcap
```

Ya los tenemos en nuestro equipo, así que vamos a comenzar viendo el contenido de **authentication.txt**

```
Hello skulloperator, as you know, we are implementing a new authentication mechanism to avoid brute-forcing...

This credential and the attached network file will be enough. I know you will get it ;)

+%7nj^g!DQxp]a>c4v&0
```

Tenemos un usuario, **skulloperator** y la contraseña **+%7nj^g!DQxp]a>c4v&0** 

Además, se habla de un nuevo mecanismo para evitar los ataques de fuerza bruta.

Utilizamos **wireshark** para abrir el archivo **network.pcap** y vemos algo interesante. Se envían trazas **icmp** a los puertos 1000, 12000 y 5000, y después, se realiza un acceso por **ssh**.

![wireshark](screenshots/wireshark.png)

Esto más la pista del archivo **authentication.txt** nos sugiere el uso de **Port Knocking** sobre el puerto 22. 

**Port Knocking** es una técnica de seguridad que permite abrir puertos en un firewall de forma dinámica mediante una secuencia previamente fijada de intentos de conexión a puertos.

Si revisamos el puerto 22 con **nmap**, aparece "filtrado", es decir, **nmap** no puede determinar si está abierto o cerrado.

![first-nmap-22](screenshots/1-nmap-22.png)

## 🔥 Explotación

Vamos a proceder a mandar trazas icmp sobre los puertos 1000, 12000 y 5000, en ese mismo orden. Para ello, podemos utilizar la herramienta **knock** o realizarlo con **bash**

```bash
knock 172.18.0.2 1000 12000 5000
```

```bash
timeout 1 bash -c "echo ''>/dev/tcp/172.18.0.2/1000" &>/dev/null
timeout 1 bash -c "echo ''>/dev/tcp/172.18.0.2/12000" &>/dev/null
timeout 1 bash -c "echo ''>/dev/tcp/172.18.0.2/5000" &>/dev/null
```

Cualquiera de estos dos métodos funcionará. 

Ahora si volvemos a revisar el puerto 22, observamos que está abierto.

![second-nmap-22](screenshots/2-nmap-22.png)

**Importante:** Si se tarda mucho en acceder al puerto 22 se cerrará y habrá que volver a usar el **Port Knocking**

## 🔑 Acceso SSH

Accedemos utilizando las credenciales encontradas anteriormente **skulloperator : +%7nj^g!DQxp]a>c4v&0**

![user](screenshots/user.png)


## 🧗 Escalada de Privilegios

### Root

Si revisamos los procesos que se están ejecutando vemos como root ejecuta un script de python.

![ps-faux](screenshots/ps-faux.png)

Si accedemos al contenido de ese archivo, vemos un servicio web que se ejecuta en el puerto 8081.

```python
import http.server
import socketserver
import urllib.parse
import subprocess
import base64
import os

PORT = 8081

AUTH_KEY_BASE64 = "d2VfYXJlX2JvbmVzXzUxMzU0NjUxNjQ4NjQ4NA=="

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        
        auth_header = self.headers.get('Authorization')

        if auth_header is None or not auth_header.startswith('Basic' ):
            self.send_response(401)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            self.wfile.write(b"Authorization header is missing or incorrect")
            return
        
        clear_text_key = auth_header.split('Basic ')[1]
        
        decoded_key = base64.b64decode(AUTH_KEY_BASE64).decode()
        
        if clear_text_key != decoded_key:
            self.send_response(403)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            self.wfile.write(b"Invalid authorization key")
            return

        parsed_path = urllib.parse.urlparse(self.path)
        query_params = urllib.parse.parse_qs(parsed_path.query)

        if 'exec' in query_params:
            command = query_params['exec'][0]
            try:
                allowed_commands = ['ls', 'whoami']
                if not any(command.startswith(cmd) for cmd in allowed_commands):
                    self.send_response(403)
                    self.send_header("Content-type", "text/plain")
                    self.end_headers()
                    self.wfile.write(b"Command not allowed.")
                    return

                result = subprocess.check_output(command, shell=True, stderr=subprocess.STDOUT)
                self.send_response(200)
                self.send_header("Content-type", "text/plain")
                self.end_headers()
                self.wfile.write(result)
            except subprocess.CalledProcessError as e:
                self.send_response(500)
                self.send_header("Content-type", "text/plain")
                self.end_headers()
                self.wfile.write(e.output)
        else:
            self.send_response(400)
            self.send_header("Content-type", "text/plain")
            self.end_headers()
            self.wfile.write(b"Missing 'exec' parameter in URL")

with socketserver.TCPServer(("", PORT), Handler) as httpd:
    httpd.serve_forever()
```

Por lo tanto, usamos **netstat** y confirmamos que este servicio se está ejecutándose en el puerto 8081 de forma interna.

![netstat](screenshots/netstat.png)

Como disponemos de credenciales para el servicio ssh, vamos a utilizarlo para aplicar **Port forwarding** y traernos ese servicio interno del puerto 8081 a nuestro puerto 8081. Hay que volver a utilizar el **Port Knocking** anterior.

```bash
ssh -L 8081:127.0.0.1:8081 skulloperator@172.18.0.2
```

De esta forma en el puerto 8081 de nuestro localhost se está ejecutando el servicio interno del puerto 8081 de la máquina.

Si realizamos un **curl** a nuestro localhost vemos que funciona y como disponemos del código de la aplicación, vamos a revisarlo.

![curl](screenshots/curl.png)

Podemos ejecutar los comandos **whoami** y **ls** proporcionando una cabecera **Authorization Basic** y una cadena de texto en base64 que debemos de decodificar.

![whoami](screenshots/whoami.png)

Como vemos, el comando se ejecuta como root. Revisando el código de la aplicación, observamos que no se aplica sanetización al comando, por lo tanto si añadimos un pipe (**|**) o un punto y coma (**;**) podremos ejecutar lo que queramos como root. Vamos a probar a usar el comando **hostname**.

![hostname](screenshots/hostname.png)

Funciona, así que vamos a darle permisos **SUID** a la bash. Para evitar problemas con la url debemos de urlencodear el comando `chmod u+s /bin/bash`

```bash
curl -s "http://localhost:8081?exec=ls|chmod%20u%2Bs%20%2Fbin%2Fbash" -H "Authorization: Basic we_are_bones_513546516486484"
```

Si volvemos a la máquina y miramos la bash, vemos que tiene permisos **SUID**, así que con `/bin/bash -p` seremos **root**.

![root](screenshots/root.png)