![[Pasted image 20260116205005.png]]

inicializar openvpn 

![[Pasted image 20260116205024.png]]

ping : El método más básico. Envías un paquete ICMP a la IP y esperas respuesta.

![[Pasted image 20260116205045.png]]

traceroute: Sirve para ver el camino hacia la IP y si responde algún salto:


```bash
traceroute 10.239.71.182
```



![[Pasted image 20260116212214.png]]

`-Pn` indica que no haga ping primero

![[Pasted image 20260116213856.png]]

![[Pasted image 20260116213906.png]]

¿Qué es Redis?

**Redis (REmote DIctionary Server)** es:

- Una **base de datos en memoria**
    
- Tipo **clave‑valor**
    
- Muy rápida
    
- Usada para **caché, sesiones, colas, tokens**, etc.

```bash
redis-cli -h <IP> -p 6379
```


no pide autenticacion

Cómo saber si conectaste bien

Si ves algo así. 10.10.11.45:6379

 Estás dentro del servicio Redis.

## foto

 
![[Pasted image 20260116214144.png]]

![[Pasted image 20260116214254.png]]


![[Pasted image 20260116214242.png]]

index 0:

![[Pasted image 20260116214307.png]]

![[Pasted image 20260116214313.png]]


## Preguntas en HTB

![[Pasted image 20260116214354.png]]

![[Pasted image 20260116214408.png]]

![[Pasted image 20260116214422.png]]

![[Pasted image 20260116214430.png]]

