#breakmyssh es una maquina de Docker labs de la categoría muy fácil.

Donde deberemos escanear la IP victima para ver que puertos nos encontramos y que tipo de vulnerabilidades hay, el titulo ya te da pistas sobre intentar conseguir el usuario y la contraseña mediante ssh.

-Descargamos el archivo de docker labs dentro de nuestra máquina controlada, donde deberá ser descomprimido mediante: 

unzip archivo.zp    


En modo root, abrimos los dos archivos con 

sudo bash auto_deploy.sh archivo.tar

y en otro terminal empezamos:

nmap a la ip básico
![[Pasted image 20241006180834.png]]

al ver un puerto abierto 22 debemos centrarnos en el 

realizamos un nmap mas exhaustivo

![[Pasted image 20241006181008.png]]


ahora lo mismo pero usando el -Pn que nos daba como tip. Pero vemos que tarda mucho en recopliar información

![[Pasted image 20241006181237.png]]



Por lo que, ya sabemos que el puerto 22 esta abierto intentaremos buscar algún tipo de versión de este servicio.

![[Pasted image 20241006181333.png]]


Encontramos la versión OpenSSH 7.7

![[Pasted image 20241006181505.png]]

Nos fijamos en user enumeration , lo explotaremos para conocer a los usuarios

serx  ~  127  searchsploit user enumeration
------------------------------------------------------------------------------------------------ ---------------------------------
 Exploit Title      



ABRIMOS METASPLOIT

msfconsole

![[Pasted image 20241006181817.png]]

ahora nos saldrá la consola de METASPLOIT

msf6 > 

aqui introduciremos otra vez el searchsploit user enumeration

msf > search user enumeration


![[Pasted image 20241006182047.png]]

Nos encontramos con uno de SSH USERNAME ENUMERATION


EJECUTAMOS UN  SEARCH SSH USER ENUMERATION para que nos salgan menos archivos


Matching Modules
================

   #  Name                                           Disclosure Date  Rank    Check  Description
   -  ----                                           ---------------  ----    -----  -----------
   0  auxiliary/scanner/ssh/cerberus_sftp_enumusers  2014-05-27       normal  No     Cerberus FTP Server SFTP Username Enumeration
   1  auxiliary/scanner/http/gitlab_user_enum        2014-11-21       normal  No     GitLab User Enumeration
   2  post/windows/gather/enum_putty_saved_sessions  .                normal  No     PuTTY Saved Sessions Enumeration Module
   3  auxiliary/scanner/ssh/ssh_enumusers            .                normal  No     SSH Username Enumeration
   4    \_ action: Malformed Packet                  .                .       .      Use a malformed packet
   5    \_ action: Timing Attack                     .                .       .      Use a timing attack


![[Pasted image 20241006182309.png]]

hacemos un Use 3 porque se encuentra en la linia 3

ahora estamos dentro
![[Pasted image 20241006182402.png]]

![[Pasted image 20241006182557.png]]
hay que poner un / antes de usr y escribir bien wordlists

ahora poner modo RUN


![[Pasted image 20241006182955.png]]


encontramos el usuario lovely

ahora usaremos #hydra para buscar y crackear la contraseña : hydra ssh://172.17.0.2 -l lovely -P /usr/share/wordlists/rockyou.txt



![[Pasted image 20241006183157.png]]

tenemos la contraseña rockyou






ahora seria realizar una conexion ssh, sabiendo el usuario, la ip y la contraseña del usuario.

pero falaria saber la contraseña de root y una vez dentro del ssh ir escalando privilegios

![[Pasted image 20241006183326.png]]

Para elevar sistemas primero es intentar buscar arhivos con permisos SUID

![[Pasted image 20241006183442.png]]

no vemos nada intereasnte, entramos en la carpeta opt

![[Pasted image 20241006183514.png]]

vemos un archivo .hash


cat .hash

y nos da un hash, hashes.com nos descrifra el hash y nos devuelve la contraseña: estrella


exit

ssh root@172.17.0.2

password: estrella

HAY OPCIONES MAS RAPIDAS, como por ejemplo usar hydra desde el principio 



![[Pasted image 20241006183822.png]]

hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2


