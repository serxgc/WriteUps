```bash
ping -c 1 {ip} #Envía **1 solo paquete ICMP** a la IP para comprobar si está accesible y medir latencia.
ping -c 1 {ip} -R #pero pide **registrar la ruta** que sigue el paquete (routers intermedios).  _Nota:_ muchos routers **bloquean o ignoran** `-R`, así que no siempre verás la ruta.
nmap -sn {ip} #Escaneo de descubrimiento (ping scan): detecta si el host está activo sin escanear puertos.

```

- Un host puede **bloquear ping** y aun así tener **servicios abiertos**.
- `nmap -sn` usa **varias técnicas** (ICMP, ARP en LAN, TCP probes), por eso es **más fiable que ping**.

Los pasos para **resolver una máquina (CTF)** son:

1. **Reconocimiento**  
    Identificar IP, puertos abiertos y servicios activos.
2. **Enumeración**  
    Analizar versiones, usuarios, rutas, configuraciones y posibles fallos.
3. **Explotación**  
    Aprovechar una vulnerabilidad para obtener acceso inicial.
4. **Post-explotación**  
    Estabilizar acceso, recolectar información interna y credenciales.
5. **Escalada de privilegios**  
    Pasar de usuario limitado a root/administrador.
6. **Demostración de compromiso**  
    Obtener la flag **si existe** o demostrar control total del sistema **si no hay flag**.


```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.129.15.254
```

![[Pasted image 20260119195218.png]]

```bash
nmap -p 3306 -sC -sV -O -vvv 10.129.15.254
nmap -p 3306 -A -vvv 10.129.15.254


```

va muy lento el scan: ya sea porque tiene algun tipo de firewall o mysql requiera autenticacion

![[Pasted image 20260119201521.png]]


![[Pasted image 20260119203149.png]]



```bash

mysql -h ip -P 3306 -u root
```
-h para el host, si no ponemos nada se ejectuaria el peurto por defecto pero no esta de mas poner el puerto abierto, ahora deberiamos pone credenciales para poder entrar, pondriamos los mas comunes ocmo admin o root. esperamos unos segundos y se nos va a conectar




![[Pasted image 20260119201710.png]]

![[Pasted image 20260119201755.png]]

Para el listamiendo de databases que hay se usa
```bash
show databases;
```
![[Pasted image 20260119201857.png]]


las tres ultimas son comunes en todas las bases de datos sql

para entrar dentro de una bases de datos
```bash
use htb
```

![[Pasted image 20260119201925.png]]
![[Pasted image 20260119202016.png]]
![[Pasted image 20260119203115.png]]


una vez estamos dentro de la base, queremos mirar el contenido, que son las tablas

```bash
show tables;
```
```bash
select * from config;
```
![[Pasted image 20260119202154.png]]

![[Pasted image 20260119203046.png]]
