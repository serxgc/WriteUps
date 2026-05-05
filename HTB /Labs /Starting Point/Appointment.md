En un **CTF**, **resolver una máquina** consiste en **analizar, comprometer y obtener control sobre el sistema objetivo**, siguiendo un proceso lógico de ataque.

Esto incluye:

- **Reconocimiento y enumeración** de servicios y posibles vulnerabilidades.
    
- **Explotación** para conseguir acceso inicial al sistema.
    
- **Escalada de privilegios** para tomar control total (root/administrador).
    

Al final, la resolución se **demuestra de dos formas posibles**:

- **Con flag**: obteniendo el archivo o dato que prueba que la máquina ha sido comprometida.
    
- **Sin flag**: demostrando acceso y control real del sistema (shell funcional, privilegios elevados, impacto verificable).

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

![[Pasted image 20260117212017.png]]

via udp

```BASH
nmap -sU --top-ports 200 -Pn <IP>


```

y nada tampoco 

![[Pasted image 20260117210943.png]]
![[Pasted image 20260117212033.png]]

![[Pasted image 20260117211040.png]]

### ACK scan (descubrir firewall)

`nmap -p- -sA -Pn <IP>`

![[Pasted image 20260117213715.png]]


![[Pasted image 20260117213706.png]]

## Qué significa `-I` en `curl`?

En realidad es **`-I` (i mayúscula)**, no `-i`.

`curl -I http://10.129.83.223`

### `-I` = **HEAD request**

- No descarga la página
    
- Solo pide **las cabeceras HTTP**
    
- Es rápido
    
- Muchos firewalls **permiten HEAD pero bloquean SYN scan**

![[Pasted image 20260117213649.png]]

![[Pasted image 20260117214056.png]]

![[Pasted image 20260117214106.png]]

![[Pasted image 20260117214116.png]]

![[Pasted image 20260117214123.png]]
