#### Challenge Scenario

All the coolest ghosts in town are going to a Haunted Houseparty - can you prove you deserve to get in?

descomprimir archivo descargado

![[Pasted image 20260108172750.png]]
Contraseña : hackthebox

$ file pass

vemos que es un ejecutale, un binario de linux
ejectuamos;

$ ./pass

![[Pasted image 20260108172852.png]]

no la sabemos la contraseña, hackthebox no es.
La categoria es reversing

vemos las string de pass

$ strings pass

y oservamos la constraseña

![[Pasted image 20260108173150.png]]

y entramos

![[Pasted image 20260108173300.png]]

podriamos usar una herramienta como ltrace

$ ltrace -s 100 ./pass

podriamos jugar con ghidra para ver el ejecutable en bajo nivel 

o usar radare 2 ./pass

$aaa
$afl
$ pdf @main