![[Pasted image 20260118093800.png]]

```bash
ping -c 1 {ip} #Envía **1 solo paquete ICMP** a la IP para comprobar si está accesible y medir latencia.
ping -c 1 {ip} -R #pero pide **registrar la ruta** que sigue el paquete (routers intermedios).  _Nota:_ muchos routers **bloquean o ignoran** `-R`, así que no siempre verás la ruta.
nmap -sn {ip} #Escaneo de descubrimiento (ping scan): detecta si el host está activo sin escanear puertos.
```

- Un host puede **bloquear ping** y aun así tener **servicios abiertos**.
- `nmap -sn` usa **varias técnicas** (ICMP, ARP en LAN, TCP probes), por eso es **más fiable que ping**.


Detecta si un host **está activo** usando **varias técnicas**, incluso cuando **ICMP está bloqueado**, **sin escanear puertos**:
```bash
nmap -sn -PS80,443 -PA80,443 -PU53 {ip}

```
- **`-sn`**  
    _No port scan_.  
    Solo **descubrimiento de host**, no busca servicios.
- **`-PS80,443`**  
    Envía **paquetes TCP SYN** a los puertos 80 y 443.  
    Si hay respuesta (SYN/ACK o RST) → **host activo**.
- **`-PA80,443`**  
    Envía **paquetes TCP ACK** a esos puertos.  
    Si responde con RST → **host activo**, incluso con firewall.
- **`-PU53`**  
    Envía **paquete UDP** al puerto 53 (DNS).  
    Si hay respuesta → **host activo**.

lanzamos nmap basico:
![[Pasted image 20260118101900.png]]

ahora uno mas agresivo:

'

ftp 
![[Pasted image 20260118102011.png]]
![[Pasted image 20260118102027.png]]


puerto 80:
![[Pasted image 20260118102637.png]]

![[Pasted image 20260118104325.png]]


puerto 21:

![[Pasted image 20260118102328.png]]

![[Pasted image 20260118104421.png]]



![[Pasted image 20260118104357.png]]

![[Pasted image 20260118104519.png]]
![[Pasted image 20260118104754.png]]

|Código|Qué hacer|
|---|---|
|200|Entrar **ya**|
|301 / 302|Seguir redirección|
|401|Pide auth → probar credenciales|
|403|Existe → buscar bypass|
|500|Error → interesante|
en firefox: http://10.129.12.110/dashboard/

encontramos un panel de login

![[Pasted image 20260118105008.png]]

paso 1. entender el login, que sale cuadno probamos credenciales erronas, no cambia url, 
el paso 2 seria entrar con burpsuite y interceptar el trafico pero como ya tenemosusuario y contraseñas, y no hay mucha cantidad podemos automatizar el proceso

```bash

hydra -L allowed.userlist -P allowed.userlist.passwd 10.129.12.110 http-post-form "/login.php:username=^USER^&password=^PASS^:F=Warning! Incorrect information."
```
F= debe cincidir con el mensasje de error
![[Pasted image 20260118105450.png]]

Son falsos positivos ya que probamos en el navegador y no funcionan, Hydra **NO sabe si entras**, solo evalúa **la respuesta HTTP** según lo que tú le dijiste en `F=`

Haz esto en el navegador:

1. Usuario inventado
2. Password inventado
    
Si:
- La página **se ve igual**
- No hay redirección
- No cambia tamaño / mensaje

➡️ Hydra **no puede distinguir éxito vs fallo** así como lo lanzaste.

abrimos burpsuite e interceptamos la señal y la enviamos al repeater.

y modificamos el request con : ![[Pasted image 20260118113424.png]]
asumiendo que es esa la contraseña
y entramos en el login y obtenemos la flag

una vez en al respuesta del repeater no nos salta el mensaje de incorrecto

![[Pasted image 20260118114432.png]]

en el request cambiamos el post por el get
GET /dashboard/index.php HTTP/1.1
Host: 10.129.12.110

