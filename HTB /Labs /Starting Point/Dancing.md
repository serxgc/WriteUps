Vamos a conectarnos como siempre através de la vpn descargada del starting point en nuestra máquina virtual. Como vemos esta maquina es un Windows, yo en máquinas windows no he tocado mucho para ser honestos, así que poco a poco iremos mejorando, seguro.

==TASK 1==: Server Message Block
==TASK 2:== 445

Empezaremos como siempre usando nmap para la enumeracion, para escanear el target una vez concectados a la vpn. 

Pero antes realizaremos el ping(ttl 127) linux ttl cercano a 64


Vamos a observar los puertos abiertos. 
`' nmap -p- --open -sS -n  -A -vvv -min--rate 5000 {ip}'` 
Recordemos que el -A  equivale a -sC -sV -O -traceroute

![[Pasted image 20251230121325.png]]

El puerto 445 tcp esta destinado al protocolo SMB, server message block, es un protocolo de red que permite compartir recursos entre equipos de una red, normalmente en entornos Windows. Corre en la capa de aplicacion o presentacion del modelo OSI

![[Pasted image 20251230121716.png]]

![[Pasted image 20251230121653.png]]

![[Pasted image 20251230121732.png]]

![[Pasted image 20251230122141.png]]


![[Pasted image 20251230122844.png]]

==TASK 3==: microsoft-ds

puerto 5985 http

puerto 139 usa netbios, antes SMB y netbios se usaban de la mano, actualmente ya no es muy común, netbioos esta practicamente obsoleto asi que con su puerto abierto podria tener vulnerabilidad.

puerto 135 es el rpc, (remote procedure call) que lo que hace es recopilar los eventos de windows, actúa como puerto de descubrimiento, el puerto 135 solo es el intermediario, con el se puede usar movimiento lateral y nunca debe ser expuesto a internet

==TASK 4:== -L (comando)

asi que vamos a listar los shares, un share es como una caja o carpeta, asi que vamos a listar esas cajas y tal vez alguna sea vulnerable.

`'smbclient -L {ip}'`
le damos a enter cuando nos pida contraseña, no hizo falta ponerla.
![[Pasted image 20251230123429.png]]

Los que tienen el símbolo dólar son como de fabrica para el smb, pero observamos otro, que este si fue creado. 

==TASK 5:== 4

==TASK 6:==
Como tal la vulnerabilidad no es del propio protocolo, si no la mala configuración de algún share. Deducimos que la respuesta es WorkShares por la cantidad de letras de la pista y por ser el unico creado, pero se podria intentar entrar en cada uno para observar si nos pide contrseña o no. En el de IPC quiza si se podria entrar sin contraseña pero es un tipo de share que no almacena archivos.

Nos conectamos a Workshares

`'smbclient \\\\{ip}\\WorkShares '`

Sin contraseña. Si nos pidiera podríamos probar las más comunes o intentar hacer un ataque de fuerza bruta.

==TASK 7:== get

`'ls'`

![[Pasted image 20251230124007.png]]

'`cd Amy.J\`'

'`ls'`

`get worknotes.txt`

`cd ..`

`cd James.P\`

`ls`

`get flag.txt`

`exit`

Observar los archivos con cat


