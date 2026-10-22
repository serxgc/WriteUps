---
tags: [ctf, forensics, atenea, sysmon, lsass, windows-eventlogs]
reto: System Monitoring
categoria: Forensics
evento: ATENEA-JORNADAS (CCN-CERT)
estado: resuelto (5/5 flags)
---

# Forensics: System Monitoring

> Este documento no es una lista de respuestas. Es el **razonamiento completo**, con **cada comando que se ejecutó, la salida real que dio y qué significa**, en el orden en que se piensa. En cada comando, la parte importante de la salida está marcada con `◀` para que veas visualmente qué te responde.

---

## 0. Cómo estudiar este documento

1. Léelo una vez entero para ver la historia.
2. Fíjate en la estructura de cada fase: **comando, salida, lectura**. Es el ritmo de trabajo.
3. Vuelve a la sección 3 (mentalidad) y a la 17 (playbook).
4. Haz la autoevaluación (sección 18) sin mirar las respuestas.
5. Repite el reto desde cero usando solo el playbook.

---

## 1. La historia en 30 segundos

El **13 de septiembre de 2024 a las 13:42:56 UTC**, alguien con una sesión de `powershell.exe` (PID 7956, usuario `DEV\developer`) lanzó:

```
rundll32.exe C:\windows\System32\comsvcs.dll MiniDump 712 C:\temp\OJBFQIVq.dmp full
```

Eso vuelca a disco la memoria del proceso 712, que es `lsass.exe`, el proceso donde Windows guarda las credenciales de los usuarios en uso. Segundos después, el mismo PowerShell abrió una conexión a `13.60.193.87:80`, una IP que no es de Microsoft. Con ese volcado, un atacante puede extraer credenciales de todos los usuarios que hubieran iniciado sesión: así se filtraron.

---

## 2. Los datos de partida

- Enunciado: brecha en septiembre de 2024, credenciales de empleados publicadas en foros; hay que analizar los logs de una estación comprometida y entender **cómo se filtraron**.
- Material: carpeta `Logs` con **148 ficheros `.evtx`** (registros de eventos de Windows).
- Objetivo: 5 flags.

| # | Pregunta |
|---|---|
| 1 | ID del proceso malicioso que accedió a un proceso remoto (ej. `1337`) |
| 2 | Nombre del proceso padre (ej. `processname.exe`) |
| 3 | PID del proceso objetivo (ej. `1337`) |
| 4 | Nombre de ese proceso (ej. `processname.exe`) |
| 5 | IP del servidor de mando y control del atacante y el puerto de origen (ej. `13.37.13.37:8000`) |

---

## 3. Mentalidad: cómo se piensa en forensics

### 3.1 Las preguntas son un mapa, no un trámite

Antes de abrir un solo log, tradúcelas a **qué tipo de evidencia necesitas**:

| Pregunta | Qué te está diciendo | Dónde buscarlo |
|---|---|---|
| "proceso malicioso que **accedió a un proceso remoto**" | Un proceso abrió la memoria de otro | Sysmon ID 10 (ProcessAccess) o la línea de comandos de un ID 1 |
| "proceso **padre**" | Hay que ver quién lanzó al malicioso | Sysmon ID 1: `ParentImage`, `ParentProcessId` |
| "PID del proceso **objetivo**" y "nombre" | Hay una víctima identificable | Argumento del comando o `TargetProcessId` / `TargetImage` |
| "IP del C2 y **puerto de origen**" | Hay una conexión de red saliente | Sysmon ID 3: `DestinationIp`, `SourcePort` |

Las preguntas ya cuentan la historia: alguien accedió a un proceso de otro, tenía un padre y habló con un servidor externo. Eso apunta a un **ataque a credenciales en memoria**.

### 3.2 Forensics es reducir ruido, en escalones

```
148 ficheros -> 3 logs relevantes -> 1 log (Sysmon) -> 21.570 eventos
   -> ~25 coincidencias -> 1 línea anómala -> 5 respuestas
```

En cada escalón la pregunta es: **¿qué puedo descartar con seguridad?** No: "¿qué miro primero?".

### 3.3 Conoce lo normal para detectar lo raro

`lsass.exe` aparece decenas de veces y es normal. `rundll32.exe` llamando a `comsvcs.dll MiniDump` no lo es. Sin conocer la línea base, todo parece sospechoso o nada lo parece.

### 3.4 Pivotar: de una pista a sus vecinos

- **Hacia atrás:** ¿quién la lanzó? (proceso padre)
- **Hacia delante:** ¿qué hizo después? (red, ficheros)
- **Lateral:** ¿qué más pasó en el mismo minuto, con el mismo usuario?

### 3.5 Verificar antes de afirmar

El comando decía `MiniDump 712`. Supuse que el 712 era LSASS, pero no lo afirmé hasta comprobarlo.

### 3.6 Una herramienta es un medio

Cuando una herramienta falló, no la arreglé: escribí cinco líneas de Python que hacían lo que necesitaba. Lo importante es tener los datos en un formato consultable.

---

## 4. Fase 1: Triage. ¿Qué logs miro?

### Comando

```bash
ls -lS | head -25
```

### Salida real

```
.rw-rw-r-- kali kali  17 MB Fri Sep 13 16:49:06 2024 Microsoft-Windows-Sysmon%4Operational.evtx   ◀ Sysmon
.rw-rw-r-- kali kali  15 MB Fri Sep 13 16:48:16 2024 Microsoft-Windows-Store%4Operational.evtx
.rw-rw-r-- kali kali  10 MB Fri Sep 13 16:48:48 2024 Security.evtx                                 ◀ Security
.rw-rw-r-- kali kali 5.0 MB Fri Sep 13 16:45:14 2024 Microsoft-Windows-AppXDeploymentServer%4Operational.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:46:30 2024 Microsoft-Windows-Storage-Storport%4Operational.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:41:10 2024 Microsoft-Windows-AppReadiness%4Admin.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:48:20 2024 Microsoft-Windows-PowerShell%4Operational.evtx  ◀ PowerShell
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:45:14 2024 Microsoft-Windows-StateRepository%4Operational.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:34:40 2024 Microsoft-Windows-Privacy-Auditing%4Operational.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:48:04 2024 Microsoft-Windows-GroupPolicy%4Operational.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:48:14 2024 System.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:48:18 2024 Microsoft-Windows-Windows Defender%4Operational.evtx  ◀ Defender
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:48:04 2024 Application.evtx
.rw-rw-r-- kali kali 1.1 MB Fri Sep 13 16:40:16 2024 Microsoft-Windows-Kernel-WHEA%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:45:14 2024 Microsoft-Windows-Winlogon%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:45:14 2024 Microsoft-Windows-ShellCommon-StartLayoutPopulation%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:45:14 2024 Microsoft-Windows-AppxPackaging%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:41:44 2024 Microsoft-Windows-LiveId%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:48:18 2024 Microsoft-Windows-PushNotification-Platform%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:48:14 2024 Microsoft-Windows-DeviceSetupManager%4Admin.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:43:18 2024 Microsoft-Windows-AppModel-Runtime%4Admin.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:46:22 2024 Microsoft-Windows-Windows Firewall With Advanced Security%4Firewall.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:48:16 2024 Microsoft-Windows-WMI-Activity%4Operational.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:45:56 2024 Microsoft-Client-Licensing-Platform%4Admin.evtx
.rw-rw-r-- kali kali 1.0 MB Fri Sep 13 16:48:14 2024 Microsoft-Windows-AppXDeployment%4Operational.evtx
```

*(Las marcas `◀` las añado yo; el comando no las imprime.)*

### Lectura

| Observación | Conclusión |
|---|---|
| Solo 4 ficheros pasan de 1,1 MB: Sysmon (17 MB), Store (15 MB), Security (10 MB), AppXDeploymentServer (5 MB) | Son los candidatos a tener volumen real de actividad |
| Decenas de ficheros pesan casi lo mismo (1,0 a 1,1 MB) | Tamaño repetido en logs muy distintos: sospecha de tamaño preasignado o por defecto, no de actividad. **Hipótesis sin confirmar**: habría que contar eventos para descartarlos |
| Fechas de modificación: **13 de septiembre de 2024**, entre las 16:34 y las 16:49 | Los logs se capturaron ese día; coincide con el enunciado (septiembre de 2024) |
| Store (15 MB) y AppXDeploymentServer (5 MB) | Volumen no es relevancia: se dejan para el final |

### Por qué Sysmon primero

Sysmon registra con mucho detalle qué ejecutable corrió, con qué argumentos, quién lo lanzó, con qué usuario y con qué hash. Para reconstruir un ataque es la mejor fuente, y el nombre del reto ("System Monitoring") lo señalaba.

### Tabla de logs: qué mirar y por qué (referencia)

| Log | Por qué mirarlo en un robo de credenciales |
|---|---|
| Sysmon Operational | Procesos (ID 1), red (3), acceso a procesos (10), ficheros (11), DNS (22) |
| PowerShell Operational y Windows PowerShell | Scripts ejecutados (ID 4104) |
| Security | Creación de procesos (4688), logons (4624, 4648), DPAPI (4692, 4693) |
| Crypto-DPAPI Operational | DPAPI protege las contraseñas guardadas del navegador |
| Windows Defender Operational | Detecciones (1116, 1117) y desactivaciones (5001, 5007) |
| Regsvr32 Operational | Ejecución de DLL con `regsvr32` |
| Bits-Client, SMBClient, WinRM, WMI-Activity, TaskScheduler | Descargas, movimiento lateral, persistencia |

---

## 5. Fase 2: Convertir el log a algo consultable

### Intento 1: la herramienta del sistema (falla)

```bash
evtx_dump -o jsonl "Microsoft-Windows-Sysmon%4Operational.evtx" > sysmon.jsonl
```

```
Traceback (most recent call last):
  File "/usr/bin/evtx_dump", line 5, in <module>
    from scripts.evtx_dump import main
ModuleNotFoundError: No module named 'scripts'
```

**Lectura:** el `evtx_dump` instalado en el sistema es un script de Python roto (no el binario de Rust que esperaba). No hace falta repararlo.

### Intento 2: comprobar qué librería tengo

```bash
python3 -c "import evtx; print(evtx.__file__)" 2>&1 || pip install evtx --break-system-packages
```

```
/home/kali/.local/lib/python3.13/site-packages/evtx/__init__.py
```

**Lectura:** la librería `evtx` ya estaba instalada. Con eso basta.

### Intento 3: un script propio de 5 líneas

```bash
cat > ~/dump.py << 'EOF'
import sys, json
from evtx import PyEvtxParser

for rec in PyEvtxParser(sys.argv[1]).records_json():
    print(json.dumps(json.loads(rec["data"])))
EOF

cd ~/Logs   # ajusta la ruta a donde tengas la carpeta
python3 ~/dump.py "Microsoft-Windows-Sysmon%4Operational.evtx" > ~/sysmon.jsonl
wc -l ~/sysmon.jsonl
```

```
cd: no such file or directory: /home/kali/Logs        ◀ error irrelevante: ya estaba dentro de Logs
21570 /home/kali/sysmon.jsonl                         ◀ 21.570 eventos
```

**Lectura:** el `cd` falló porque ya estaba en la carpeta, y el volcado funcionó igualmente. **21.570 eventos**, un evento por línea.

### Los errores de entorno (y qué enseñan)

| Qué pasó | Salida real | Lección |
|---|---|---|
| Creí que el fichero estaba vacío | `ls -l` mostró `20 MB` | **Verifica antes de concluir**; lo que faltaba era el script `q.py`, no los datos |
| Intento de ejecutar un script que no existía | `can't open file '/home/kali/q.py': [Errno 2] No such file or directory` | Lee el error: era un fichero no creado, no un problema de datos |
| `jq` sin instalar | `Command 'jq' not found, but can be installed with: sudo apt install jq` | Si falta una herramienta, usa un script propio |

```bash
ls -l ~/sysmon.jsonl
```

```
.rw-rw-r-- kali kali 20 MB Wed Oct 21 08:08:58 2026  /home/kali/sysmon.jsonl     ◀ 20 MB, no estaba vacío
```

Un detalle técnico que explica el riesgo: el `>` **vacía el fichero destino antes de ejecutar** el comando. Si relanzas un volcado desde otra carpeta y falla, pierdes el fichero. Usa rutas absolutas.

---

## 6. Fase 3: Mapa del log

### Comando

Primero se crea el script `q.py` (cuenta eventos por ID o busca patrones):

```bash
cat > ~/q.py << 'EOF'
import sys, json, re, collections

pat = re.compile(sys.argv[2], re.I) if len(sys.argv) > 2 else None
cnt = collections.Counter()

for line in open(sys.argv[1]):
    e = json.loads(line)["Event"]
    eid = e["System"]["EventID"]
    if isinstance(eid, dict):
        eid = eid.get("#text")
    if not pat:
        cnt[eid] += 1
        continue
    if pat.search(line):
        d = e.get("EventData") or {}
        t = e["System"]["TimeCreated"]["#attributes"]["SystemTime"]
        extra = d.get("CommandLine") or d.get("TargetFilename") or d.get("TargetObject") or ""
        print(t, eid, d.get("Image", ""), extra, sep=" | ")

if not pat:
    for k, v in cnt.most_common():
        print(v, k)
EOF

python3 ~/q.py ~/sysmon.jsonl
```

### Salida real

```
11678 255     ◀ errores internos de Sysmon (se descartan)
3059 11       ◀ ficheros creados
2826 13       ◀ registro modificado
1697 1        ◀ PROCESOS CREADOS (el más importante)
1357 5
388 7
361 12
56 26
53 29
27 17
20 22         ◀ consultas DNS
15 2
14 10         ◀ acceso a procesos (la pregunta 1)
7 3           ◀ conexiones de red (la pregunta 5)
5 4
3 6
2 16
2 8
```

### Lectura: qué significa cada ID

| ID | Eventos | Significado en Sysmon | ¿Útil aquí? |
|---|---|---|---|
| 255 | 11.678 | Error interno de Sysmon | No, se descarta |
| 11 | 3.059 | Fichero creado | Sí, para ver el `.dmp` |
| 13 | 2.826 | Valor de registro modificado | Poco |
| 1 | 1.697 | **Proceso creado** | **Sí, el más importante** |
| 5 | 1.357 | Proceso terminado | Poco |
| 7 | 388 | DLL cargada | Algo |
| 12 | 361 | Objeto de registro creado/borrado | Poco |
| 26 | 56 | Fichero borrado (registrado) | A mirar si hay limpieza |
| 29 | 53 | Ejecutable detectado | Posible |
| 17 | 27 | Pipe creado | Poco |
| 22 | 20 | **Consulta DNS** | Sí, dominios |
| 2 | 15 | Hora de creación de fichero cambiada | A mirar (anti-forense) |
| 10 | 14 | **Acceso a un proceso** | **Sí, pregunta 1** |
| 3 | 7 | **Conexión de red** | **Sí, pregunta 5** |
| 4, 6, 16, 8 | pocos | Estado, driver, configuración, hilo remoto | Marginal |

### Por qué este paso

Antes de buscar, quería saber cuánto material hay de cada tipo. Hay **7 conexiones de red** y **14 accesos a procesos**: son pocos, se pueden leer enteros. Hay **1.697 creaciones de proceso**: demasiadas para leerlas todas, así que hay que buscar con criterio.

---

## 7. Fase 4: Hipótesis y búsqueda dirigida

### La hipótesis

Enunciado ("credenciales publicadas"), estación comprometida y preguntas ("proceso que accedió a otro proceso") → **robo de credenciales en memoria**; el objetivo casi seguro es `lsass.exe`.

### Comando

```bash
python3 ~/q.py ~/sysmon.jsonl 'lsass|mimikatz|procdump|comsvcs|Login Data|Local State|logins\.json|key4\.db|\.kdbx|webhook|pastebin|telegram' | head -50
```

### Por qué esos patrones

| Patrón | Qué sugiere |
|---|---|
| `lsass`, `mimikatz`, `procdump`, `comsvcs` | Volcado de memoria de credenciales de Windows |
| `Login Data`, `Local State` | Contraseñas guardadas de Chrome/Edge |
| `logins.json`, `key4.db` | Contraseñas de Firefox |
| `.kdbx` | Base de datos de un gestor de contraseñas |
| `webhook`, `pastebin`, `telegram` | Exfiltración |

### Salida real

```
2024-05-17T15:45:07.505740Z | 1 | C:\Windows\System32\lsass.exe | C:\Windows\system32\lsass.exe
2024-09-13T12:20:43.087948Z | 1 | C:\Windows\System32\lsass.exe | C:\Windows\system32\lsass.exe
2024-09-13T13:13:55.528390Z | 1 | C:\Windows\System32\lsass.exe | C:\Windows\system32\lsass.exe
2024-09-13T13:25:41.695047Z | 11 | C:\Windows\system32\lsass.exe | C:\Windows\System32\Microsoft\Protect\S-1-5-18\c1d29ba4-2e60-4e2a-b8da-975194348b20
2024-09-13T13:33:41.652876Z | 11 | C:\Windows\System32\poqexec.exe | C:\Windows\WinSxS\Backup\amd64_microsoft-windows-lsa-minwin_31bf3856ad364e35_10.0.19041.4239_none_b22e359f4782b59b_lsass.exe_682060de
2024-09-13T13:35:24.179709Z | 1 | C:\Windows\System32\lsass.exe | C:\Windows\system32\lsass.exe
2024-09-13T13:35:24.179867Z | 10 |  | 
2024-09-13T13:35:24.179993Z | 10 |  | 
2024-09-13T13:35:24.180865Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\LsaPid
2024-09-13T13:35:24.182292Z | 11 | C:\Windows\system32\lsass.exe | C:\Windows\debug\PASSWD.LOG
2024-09-13T13:35:24.184579Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Name
2024-09-13T13:35:24.184684Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Comment
2024-09-13T13:35:24.184823Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Capabilities
2024-09-13T13:35:24.184932Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\RpcId
2024-09-13T13:35:24.185036Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Version
2024-09-13T13:35:24.185142Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\TokenSize
2024-09-13T13:35:24.185242Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Time
2024-09-13T13:35:24.185344Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Type
2024-09-13T13:35:24.185432Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\Time
2024-09-13T13:35:24.185698Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\ProductType
2024-09-13T13:35:24.206556Z | 17 | C:\Windows\system32\lsass.exe | 
2024-09-13T13:42:56.218249Z | 1 | C:\Windows\System32\rundll32.exe | "C:\Windows\system32\rundll32.exe" C:\windows\System32\comsvcs.dll MiniDump 712 C:\temp\OJBFQIVq.dmp full     ◀◀◀ LA LÍNEA ANÓMALA
2024-09-13T13:42:56.309768Z | 7 | C:\Windows\System32\rundll32.exe |
```

### Lectura: separar ruido de señal

| Líneas | Por qué son normales |
|---|---|
| ID 1 de `lsass.exe` a las 12:20, 13:13 y 13:35 | El sistema arranca `lsass` en cada inicio. Hay **tres arranques** en el log |
| Ruta `C:\Windows\System32\lsass.exe` | Ruta legítima. Un `lsass.exe` en otra carpeta sería sospechoso |
| IDs 13 y 11 de `lsass` a las 13:35:24 (`SspiCache`, `PASSWD.LOG`, `LsaPid`) | Actividad interna normal de LSASS al iniciarse |
| ID 10 de las 13:35:24 (sin campos en esta vista) | Accesos del sistema al arrancar; se ven completos en la fase 8 |

La línea marcada es la única que **no encaja**:

| Elemento | Por qué es sospechoso |
|---|---|
| `rundll32.exe` | Programa legítimo que ejecuta funciones de DLL. Los atacantes lo usan porque **no levanta tantas alarmas como un ejecutable propio** (técnica *living off the land*) |
| `comsvcs.dll` + `MiniDump` | Función pensada para volcar memoria de procesos. Nadie la llama a mano en el uso diario |
| `712` | El proceso a volcar |
| `C:\temp\OJBFQIVq.dmp` | Destino del volcado: carpeta de escritura libre, nombre aleatorio |
| `full` | Volcado completo de memoria |

### Qué aprendo

Una búsqueda por patrones da **candidatos**, no respuestas. Lo que importa es **descartar con argumentos** (ruta, momento, proceso, actividad) y no por intuición. Y la anomalía está en la **combinación** (`rundll32` + `comsvcs.dll` + `MiniDump`), no en una pieza suelta.

---

## 8. Fase 5: Pivotar. Pedir el evento completo

### Comandos

```bash
cat > ~/r.py << 'EOF'
import sys, json
f = sys.argv[1]
eids = set(sys.argv[2].split(","))
for line in open(f):
    e = json.loads(line)["Event"]
    eid = e["System"]["EventID"]
    eid = eid.get("#text") if isinstance(eid, dict) else eid
    if str(eid) not in eids:
        continue
    t = e["System"]["TimeCreated"]["#attributes"]["SystemTime"]
    print(t, eid, json.dumps(e.get("EventData") or {}))
EOF

python3 ~/r.py ~/sysmon.jsonl 1 | grep -i comsvcs
```

### Salida real (en una sola línea JSON, reformateada aquí campo por campo para leerla)

```
Hora de registro     : 2024-09-13T13:42:56.218249Z
RuleName             : technique_id=T1218.011,technique_name=rundll32.exe
UtcTime              : 2024-09-13 13:42:56.216
ProcessGuid          : E4EA3175-4160-66E4-2D01-000000000800
ProcessId            : 8996                                              ◀ FLAG 1
Image                : C:\Windows\System32\rundll32.exe
FileVersion          : 10.0.19041.4648 (WinBuild.160101.0800)
Description          : Windows host process (Rundll32)
Product              : Microsoft® Windows® Operating System
Company              : Microsoft Corporation
OriginalFileName     : RUNDLL32.EXE
CommandLine          : "C:\Windows\system32\rundll32.exe" C:\windows\System32\comsvcs.dll MiniDump 712 C:\temp\OJBFQIVq.dmp full
                                                                          ◀ objetivo 712 (FLAGS 3 y 4)
CurrentDirectory     : C:\Windows\system32\
User                 : DEV\developer
LogonGuid            : E4EA3175-4001-66E4-906C-160000000000
LogonId              : 0x166c90
TerminalSessionId    : 1
IntegrityLevel       : High
Hashes               : SHA1=34661F530C6BC94AA1F307DF30F75733E5D87382,
                       MD5=D28778D07C8F7CA59B7569E4EDA54512,
                       SHA256=770832DA77324F205306B4D89C02BA2B98DCE87207A82D4BF9B1D076608862D6,
                       IMPHASH=F207A867D6098266EE63A3CA677EB0FF
ParentProcessGuid    : E4EA3175-4154-66E4-2B01-000000000800
ParentProcessId      : 7956                                              ◀ PID del padre (lo necesitaremos en la red)
ParentImage          : C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe     ◀ FLAG 2
ParentCommandLine    : "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
ParentUser           : DEV\developer
```

### Lectura campo a campo

| Campo | Valor | Qué me dice |
|---|---|---|
| `ProcessId` | 8996 | **Flag 1**: el proceso que hace el volcado |
| `Image` | `rundll32.exe` | El binario legítimo usado como herramienta |
| `CommandLine` | `... MiniDump 712 C:\temp\OJBFQIVq.dmp full` | Qué hizo exactamente: objetivo 712, destino `C:\temp\OJBFQIVq.dmp` |
| `User` | `DEV\developer` | Se ejecutó con la cuenta de un usuario real, no con SYSTEM |
| `IntegrityLevel` | `High` | Privilegios elevados, necesarios para acceder a LSASS |
| `ParentImage` | `...\powershell.exe` | **Flag 2** |
| `ParentProcessId` | 7956 | Identifica al PowerShell concreto; se reutiliza al mirar la red |
| `ParentUser` | `DEV\developer` | El padre corre con el mismo usuario |
| `RuleName` | `T1218.011` | Sysmon ya lo clasifica como ejecución mediante rundll32 |
| `Hashes` | SHA256 `770832DA...` | Permite comprobar si es el `rundll32` legítimo |

### Por qué importa

Un solo evento responde a **dos preguntas** (1 y 2) y da la mayor parte del contexto: quién, con qué privilegios, desde qué proceso, con qué comando. El pivote **hacia atrás** (de hijo a padre) convierte "alguien ejecutó algo" en "un PowerShell del usuario `developer` lo ejecutó".

### Qué aprendo

Cuando una línea resumida te llama la atención, **pide el evento completo**. Los campos que no mostrabas contienen la mayoría de las respuestas.

---

## 9. Fase 6: Verificar el objetivo (PID 712)

### Comando

```bash
python3 ~/q.py ~/sysmon.jsonl 'ProcessId": ?712' | head
```

### Salida real

```
2024-09-13T13:35:24.179709Z | 1 | C:\Windows\System32\lsass.exe | C:\Windows\system32\lsass.exe      ◀ el PID 712 es lsass.exe
2024-09-13T13:35:24.179867Z | 10 |  | 
2024-09-13T13:35:24.179993Z | 10 |  | 
2024-09-13T13:35:24.180865Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\LsaPid
2024-09-13T13:35:24.182292Z | 11 | C:\Windows\system32\lsass.exe | C:\Windows\debug\PASSWD.LOG
2024-09-13T13:35:24.184579Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Name
2024-09-13T13:35:24.184684Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Comment
2024-09-13T13:35:24.184823Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Capabilities
2024-09-13T13:35:24.184932Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\RpcId
2024-09-13T13:35:24.185036Z | 13 | C:\Windows\system32\lsass.exe | HKLM\System\CurrentControlSet\Control\Lsa\SspiCache\credssp.dll\Version
```

### Lectura

Todos los eventos con `ProcessId` 712 son de `C:\Windows\System32\lsass.exe`, creado en el **arranque de las 13:35:24**. El volcado ocurre a las **13:42:56**, unos **7 minutos** después y sin otro arranque en medio (los tres arranques del log son 12:20, 13:13 y 13:35), así que el PID no cambió. Esto valida las **flags 3 (`712`) y 4 (`lsass.exe`)**.

### Por qué verificar

Un PID solo identifica un proceso **mientras dura**: tras un reinicio, el mismo número puede ser otro proceso. Con tres arranques en el log, comprobar que el 712 era `lsass` **en el arranque inmediatamente anterior** al volcado es lo que da validez a la respuesta.

### Qué aprendo

Cualquier identificador efímero (PID, handle, puerto de origen) hay que **anclarlo en el tiempo**: ¿qué era ese número en ese momento?

---

## 10. Fase 7: ¿Hay un evento de acceso (ID 10)?

### Qué esperaba

La pregunta 1 habla de "accedió a un proceso remoto". Esperaba un ID 10 con origen `rundll32.exe` y destino `lsass.exe`.

### Comando

```bash
python3 ~/r.py ~/sysmon.jsonl 10 | grep -i lsass
```

### Salida real

```
2024-09-13T13:35:24.179867Z 10 {"RuleName": "technique_id=T1003,technique_name=Credential Dumping", "UtcTime": "2024-09-13 13:35:18.865", "SourceProcessGUID": "E4EA3175-3F96-66E4-0800-000000000800", "SourceProcessId": 548, "SourceThreadId": 552, "SourceImage": "C:\\Windows\\system32\\wininit.exe", "TargetProcessGUID": "E4EA3175-3F96-66E4-0C00-000000000800", "TargetProcessId": 712, "TargetImage": "C:\\Windows\\system32\\lsass.exe", "GrantedAccess": "0x1fffff", "CallTrace": "C:\\Windows\\SYSTEM32\\ntdll.dll+9ee74|C:\\Windows\\SYSTEM32\\KERNELBASE.dll+53815|C:\\Windows\\SYSTEM32\\KERNELBASE.dll+50856|C:\\Windows\\SYSTEM32\\KERNEL32.DLL+1cef4|C:\\Windows\\system32\\wininit.exe+27813|C:\\Windows\\system32\\wininit.exe+25508|C:\\Windows\\system32\\wininit.exe+29c41|C:\\Windows\\system32\\wininit.exe+2ad6f|C:\\Windows\\system32\\wininit.exe+48a6|C:\\Windows\\SYSTEM32\\KERNEL32.DLL+17374|C:\\Windows\\SYSTEM32\\ntdll.dll+4cc91", "SourceUser": "NT AUTHORITY\\SYSTEM", "TargetUser": "NT AUTHORITY\\SYSTEM"}
2024-09-13T13:35:24.179993Z 10 {"RuleName": "technique_id=T1003,technique_name=Credential Dumping", "UtcTime": "2024-09-13 13:35:18.865", "SourceProcessGUID": "E4EA3175-3F96-66E4-0600-000000000800", "SourceProcessId": 472, "SourceThreadId": 620, "SourceImage": "C:\\Windows\\system32\\csrss.exe", "TargetProcessGUID": "E4EA3175-3F96-66E4-0C00-000000000800", "TargetProcessId": 712, "TargetImage": "C:\\Windows\\system32\\lsass.exe", "GrantedAccess": "0x1fffff", "CallTrace": "C:\\Windows\\SYSTEM32\\ntdll.dll+9da24|C:\\Windows\\system32\\basesrv.DLL+326f|C:\\Windows\\SYSTEM32\\CSRSRV.dll+6490|C:\\Windows\\SYSTEM32\\ntdll.dll+4cc9f", "SourceUser": "NT AUTHORITY\\SYSTEM", "TargetUser": "NT AUTHORITY\\SYSTEM"}
```

### Lectura

| Origen | PID origen | Destino | PID destino | Acceso | Usuario |
|---|---|---|---|---|---|
| `wininit.exe` | 548 | `lsass.exe` | **712** | `0x1fffff` | SYSTEM |
| `csrss.exe` | 472 | `lsass.exe` | **712** | `0x1fffff` | SYSTEM |

- `0x1fffff` es `PROCESS_ALL_ACCESS` (control total). Suena alarmante, pero son procesos del sistema (`wininit`, `csrss`) a las **13:35:18, durante el arranque**: es normal. Sysmon los etiqueta como "Credential Dumping" porque su regla salta ante cualquier acceso a LSASS, sea legítimo o no.
- Estos dos eventos **confirman otra vez** que el 712 es `lsass.exe` (campo `TargetImage`).
- **No hay ningún acceso registrado desde `rundll32`.**

### Cómo interpreto la ausencia

Esto **no invalida** la respuesta, pero es un matiz que hay que saber:

- El volcado se deduce de la **línea de comandos** del ID 1, que es inequívoca; no de un evento de acceso.
- Sysmon solo registra el ID 10 según su configuración. No sé con certeza por qué no aparece.
- Por eso la flag 1 se responde con el `ProcessId` del `rundll32` (**8996**), el proceso que "accedió" al proceso remoto según su comando.

### Qué aprendo

**Distingue lo que ves de lo que deduces.** Aquí vi un comando; deduzco un acceso. Un informe profesional marca esa diferencia.

---

## 11. Fase 8: La red (pregunta 5)

### Comando

Solo hay 7 conexiones, así que se leen **todas**:

```bash
python3 ~/r.py ~/sysmon.jsonl 3
```

### Salida real (las 7 líneas, reformateadas en tabla; todas son TCP, `Initiated: true`, origen `10.0.2.15`)

| # | Hora registro (UTC) | UtcTime | Proceso (PID) | Origen | Destino |
|---|---|---|---|---|---|
| 1 | 13:27:58 | 13:28:12 | `MpDefenderCoreService.exe` (2792) | `10.0.2.15:50373` | `52.113.194.132:443` |
| 2 | 13:35:37 | 13:35:25 | `MpDefenderCoreService.exe` (2884) | `10.0.2.15:49674` | `52.182.143.213:443` |
| 3 | 13:37:09 | 13:37:12 | `MsMpEng.exe` (2968) | `10.0.2.15:49694` | `20.166.109.4:443` |
| 4 | 13:42:34 | 13:42:52 | `MsMpEng.exe` (2968) | `10.0.2.15:49772` | `20.166.109.4:443` |
| 5 | 13:42:48 | 13:43:07 | `MpDefenderCoreService.exe` (2884) | `10.0.2.15:49775` | `52.182.143.213:443` |
| **6** | **13:42:58** | **13:43:15** | **`powershell.exe` (7956)** | **`10.0.2.15:49776`** | **`13.60.193.87:80`** |
| 7 | 13:45:50 | 13:46:00 | `MpDefenderCoreService.exe` (6724) | `10.0.2.15:49789` | `20.42.65.84:443` |

La conexión importante, tal cual salió:

```
2024-09-13T13:42:58.988857Z 3 {"RuleName": "technique_id=T1059.001,technique_name=PowerShell", "UtcTime": "2024-09-13 13:43:15.965", "ProcessGuid": "E4EA3175-4154-66E4-2B01-000000000800", "ProcessId": 7956, "Image": "C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\powershell.exe", "User": "DEV\\developer", "Protocol": "tcp", "Initiated": true, "SourceIsIpv6": false, "SourceIp": "10.0.2.15", "SourceHostname": "-", "SourcePort": 49776, "SourcePortName": "-", "DestinationIsIpv6": false, "DestinationIp": "13.60.193.87", "DestinationHostname": "-", "DestinationPort": 80, "DestinationPortName": "-"}
                       ProcessId = 7956 ◀ el MISMO PID que el padre del rundll32
                                                          SourcePort = 49776 ◀ FLAG 5 (puerto de origen)
                                                                          DestinationIp = 13.60.193.87 ◀ FLAG 5 (IP del C2)
```

### Por qué la del PowerShell destaca

| Criterio | Defender (normal) | PowerShell (anómala) |
|---|---|---|
| Proceso | Servicio de seguridad de Windows | Intérprete de comandos, que normalmente no abre conexiones externas |
| Puerto de destino | 443 (HTTPS, cifrado) | 80 (HTTP, sin cifrar) |
| Destino | Rangos de Microsoft (`20.x`, `52.x`) | `13.60.193.87`, fuera de ese patrón |
| Relación con el incidente | Ninguna | **Es el padre del `rundll32` (PID 7956)** y conecta **justo después** del volcado |

Dos pistas independientes (proceso y destino) apuntan a lo mismo, y el **PID 7956** une la conexión de red con la cadena de procesos de la fase 5.

### Por qué "puerto de origen" y no el de destino

`DestinationPort` es 80 (el del servidor). `SourcePort` es **49776**, un puerto efímero que el sistema asigna a la conexión saliente. La pregunta lo pide explícitamente, así que la respuesta es `13.60.193.87:49776` y no `:80`.

### Una nota de rigor sobre las horas

En la tabla se ve que la **hora de registro** y el **`UtcTime`** del propio evento difieren unos segundos (por ejemplo, en la conexión 6: 13:42:58 frente a 13:43:15). Para ordenar los hechos me fío del `UtcTime`, que sitúa la conexión **después** del volcado (13:42:56). En un informe se anota la discrepancia, no se ignora.

### Qué aprendo

Cuando hay **pocos eventos**, léelos todos. Y para decidir cuál es el anómalo, compara en varias dimensiones (proceso, puerto, destino, momento, relación con el incidente), no en una sola.

---

## 12. Las respuestas, con su evidencia

| # | Pregunta | Respuesta | Evidencia (de qué comando salió) |
|---|---|---|---|
| 1 | PID del proceso malicioso | `8996` | `r.py ... 1 \| grep comsvcs`, campo `ProcessId` |
| 2 | Proceso padre | `powershell.exe` | Mismo evento, campo `ParentImage` |
| 3 | PID del objetivo | `712` | Argumento de `MiniDump`; verificado con `q.py ... 'ProcessId": ?712'` |
| 4 | Nombre del objetivo | `lsass.exe` | Mismo comando de verificación y `TargetImage` del ID 10 |
| 5 | C2 y puerto de origen | `13.60.193.87:49776` | `r.py ... 3`, evento del `powershell.exe` PID 7956 |

### Trampas en las que se puede caer

- **Flag 1:** confundir 8996 (el malicioso) con 712 (la víctima) o con 7956 (el padre). Si la plataforma rechazara el 8996, el siguiente candidato lógico sería 7956.
- **Flag 5:** poner el puerto 80 (destino) en vez del 49776 (origen).
- **Zona horaria:** las horas de Sysmon están en UTC.
- **Ruido de Defender:** parecen "conexiones externas", pero van a Microsoft por el 443.

---

## 13. Línea de tiempo

| Hora (UTC, 13/09/2024) | Evento | Fuente |
|---|---|---|
| 13:35:18 a 13:35:24 | Arranque; `lsass.exe` (PID 712) se inicia; `wininit` y `csrss` lo acceden (normal) | ID 1 y 10 |
| 13:42:56 | `powershell.exe` (7956) lanza `rundll32.exe` (8996) con `comsvcs.dll MiniDump 712 ...` | ID 1 |
| 13:43:15 (UtcTime) | `powershell.exe` (7956) conecta a `13.60.193.87:80` desde el puerto 49776 | ID 3 |

---

## 14. Cadena del ataque y técnicas MITRE ATT&CK

| Fase | Qué ocurre | Técnica |
|---|---|---|
| Ejecución | Sesión de PowerShell como `DEV\developer` | T1059.001 (PowerShell) |
| Evasión | `rundll32.exe` como proxy para ejecutar la DLL | T1218.011 (Rundll32) |
| Acceso a credenciales | Volcado de memoria de LSASS con `comsvcs.dll` | T1003.001 (LSASS Memory) |
| Mando y control / exfiltración | PowerShell conecta a una IP externa por HTTP | T1071.001 (probable; no se ve el contenido) |

Acceso inicial: **no determinado** con estos datos.

---

## 15. Lo que NO sabemos (y cómo seguir)

| Pregunta abierta | Dónde investigarlo |
|---|---|
| ¿Cómo entró el atacante? | Security (4624 logons), RDP/WinRM, descargas |
| ¿Qué comandos ejecutó el PowerShell? | `Microsoft-Windows-PowerShell%4Operational` (ID 4104) y `Windows PowerShell` |
| ¿Qué se envió a `13.60.193.87`? | No hay captura de red; solo se ve la conexión |
| ¿Qué pasó con `C:\temp\OJBFQIVq.dmp`? | Sysmon ID 11 (creación) y 26 (borrado) |
| ¿Se intentó desactivar Defender? | Defender Operational (5001, 5007) |
| ¿Por qué no hay un ID 10 desde `rundll32`? | Configuración de Sysmon |

Ejercicio recomendado: responde a las dos primeras tú solo con el playbook.

---

## 16. Conclusión

Las credenciales se filtraron porque un atacante con una sesión de PowerShell en la estación volcó la memoria de LSASS con binarios de Windows y, a continuación, el mismo PowerShell contactó con un servidor externo. Con el volcado se pueden extraer credenciales de todos los usuarios con sesión en esa máquina.

La evidencia es sólida en tres puntos (comando de volcado, proceso padre, conexión posterior) y tiene lagunas reconocidas en otros dos (acceso inicial y contenido enviado).

---

## 17. Playbook reutilizable (cualquier reto de logs de Windows)

1. **Lee las preguntas** y tradúcelas a tipos de evidencia (proceso, red, fichero, usuario, tiempo).
2. **Elige el log por la evidencia que necesitas**; el tamaño solo orienta.
3. **Comprueba qué herramientas tienes** (`which`) y convierte el log a texto consultable.
4. **Cuenta los eventos por tipo.** Pocos: léelos todos. Muchos: filtra.
5. **Formula una hipótesis** a partir del enunciado.
6. **Busca patrones de esa hipótesis** y trata el resultado como **candidatos**.
7. **Descarta con argumentos** (ruta, momento, padre, usuario).
8. **Pivota** sobre el evento sospechoso: padre, hijos, red, ficheros.
9. **Pide el evento completo** y lee todos los campos.
10. **Verifica los identificadores** (PID, puertos) anclándolos en el tiempo.
11. **Anota qué sabes y qué deduces.** Marca las lagunas.
12. **Ordena una línea de tiempo** y escribe la cadena del ataque.

---

## 18. Autoevaluación (responde sin mirar)

> [!question]- 1. ¿Por qué `lsass.exe` aparece muchas veces en la búsqueda y no es sospechoso?
> Porque el sistema lo arranca en cada inicio y lo usa constantemente. Lo normal se reconoce por la ruta (`System32`), el momento (arranque) y su actividad interna (registro, `PASSWD.LOG`). Lo sospechoso es quién **lo accede** y cómo.

> [!question]- 2. ¿Qué hace `rundll32 comsvcs.dll MiniDump 712 C:\temp\x.dmp full`?
> Ejecuta la función `MiniDump` de `comsvcs.dll` para volcar a disco la memoria del proceso 712 en `C:\temp\x.dmp`. Aquí el 712 es LSASS, así que el fichero contiene credenciales.

> [!question]- 3. ¿Por qué se usa `rundll32` y `comsvcs.dll` en vez de Mimikatz?
> Son componentes legítimos de Windows ("living off the land"), por lo que levantan menos sospechas que un ejecutable de hacking conocido.

> [!question]- 4. ¿Qué diferencia hay entre `SourcePort` y `DestinationPort` y cuál pide la pregunta 5?
> `DestinationPort` es el puerto del servidor (80). `SourcePort` es el puerto temporal que el equipo usa para salir (49776). La pregunta pide el de origen.

> [!question]- 5. ¿Por qué hay que comprobar que el PID 712 era LSASS y no asumirlo?
> Porque un PID solo identifica a un proceso mientras dura; con reinicios, el mismo número puede ser otro proceso. Se verificó en el arranque inmediatamente anterior al volcado.

> [!question]- 6. ¿Qué dato demuestra que el PowerShell conectado a `13.60.193.87` es el mismo que lanzó el volcado?
> El PID: el `ParentProcessId` del `rundll32` es 7956, y la conexión de red es del `ProcessId` 7956.

> [!question]- 7. ¿Por qué los dos accesos a LSASS con `0x1fffff` no son el ataque?
> Los hacen `wininit.exe` y `csrss.exe`, procesos del sistema, durante el arranque (13:35:18), unos 7 minutos antes del volcado. Sysmon los marca como "Credential Dumping" porque su regla salta ante cualquier acceso a LSASS.

> [!question]- 8. ¿Por qué no hay un ID 10 desde `rundll32` hacia LSASS?
> No se sabe con certeza. Sysmon solo registra lo que su configuración le indica. La prueba del volcado es la línea de comandos del ID 1.

> [!question]- 9. ¿Qué dejarías para investigar después?
> Cómo consiguió el atacante la sesión de PowerShell, qué se envió a `13.60.193.87`, qué ejecutó el PowerShell (log 4104) y si el `.dmp` se borró o se comprimió.

---

## 19. Glosario

| Término | Significado |
|---|---|
| **LSASS** | Proceso de Windows que guarda las credenciales de los usuarios en memoria |
| **PID** | Número que identifica un proceso mientras dura |
| **Sysmon** | Herramienta de Microsoft que registra procesos, red y ficheros con mucho detalle |
| **LOLBin / Living off the land** | Uso de programas legítimos del sistema para fines maliciosos |
| **C2** | Servidor de mando y control del atacante |
| **Puerto efímero** | Puerto temporal que el sistema asigna a una conexión saliente |
| **EVTX** | Formato binario de los registros de eventos de Windows |
| **Proceso padre** | El proceso que lanzó a otro |
| **UtcTime** | Hora del propio evento según Sysmon, en UTC |

---

## 20. Anatomía de los scripts: para que puedas escribirlos tú

Los tres scripts (`dump.py`, `q.py`, `r.py`) no son magia ni el análisis en sí: son **lupas**. El análisis (qué buscar, qué descartar, qué significa) lo haces tú. Esta sección explica cada línea para que puedas escribirlos de memoria y, sobre todo, adaptarlos.

### 20.1 La forma de un evento (lo primero que hay que entender)

Cada línea de `sysmon.jsonl` es un evento con esta estructura (resumida):

```
{
  "Event": {
    "System": {                          <- metadatos: qué tipo de evento y cuándo
      "EventID": 1,
      "TimeCreated": { "#attributes": { "SystemTime": "2024-09-13T13:42:56.218249Z" } },
      ...
    },
    "EventData": {                       <- el contenido: cambia según el tipo de evento
      "ProcessId": 8996,
      "Image": "C:\\Windows\\System32\\rundll32.exe",
      "CommandLine": "...",
      "ParentImage": "...",
      ...
    }
  }
}
```

**Cómo descubrí los nombres de los campos:** no los sabía de memoria. Imprimí un evento completo (con `r.py`) y los leí. Esa es la técnica: *no memorices nombres de campos, imprime un evento y léelo.*

Para que lo practiques sin script, mira un evento a mano:

```bash
head -1 ~/sysmon.jsonl | python3 -m json.tool | head -40
```

`python3 -m json.tool` reformatea un JSON para que sea legible. Compruébalo.

### 20.2 `dump.py`, línea a línea

```python
import sys, json
from evtx import PyEvtxParser

for rec in PyEvtxParser(sys.argv[1]).records_json():
    print(json.dumps(json.loads(rec["data"])))
```

| Línea | Qué hace | Por qué |
|---|---|---|
| `import sys, json` | Carga dos módulos de Python: `sys` (acceso a los argumentos del programa) y `json` (leer y escribir JSON) | Hacen falta los dos |
| `from evtx import PyEvtxParser` | Carga la clase que sabe leer el formato binario `.evtx` | Sin ella, un `.evtx` es una ristra de bytes ilegible |
| `sys.argv[1]` | El primer argumento que pasas al script (`argv[0]` es el nombre del propio script) | Así eliges el fichero al ejecutarlo |
| `PyEvtxParser(...)` | Abre ese fichero | |
| `.records_json()` | Recorre los eventos uno a uno; cada `rec` es un diccionario cuyo campo `data` contiene el evento **como texto JSON** | |
| `json.loads(rec["data"])` | Convierte ese texto en una estructura de Python | |
| `json.dumps(...)` | La vuelve a convertir en texto, pero **en una sola línea** | Queremos un evento por línea para poder usar `grep` |
| `print(...)` | Escribe la línea en pantalla | Con `> fichero` en la terminal se guarda en un fichero |

Idea para recordar: el script **lee eventos y los escribe uno por línea**. Nada más.

### 20.3 `q.py`, línea a línea

```python
import sys, json, re, collections

pat = re.compile(sys.argv[2], re.I) if len(sys.argv) > 2 else None
cnt = collections.Counter()

for line in open(sys.argv[1]):
    e = json.loads(line)["Event"]
    eid = e["System"]["EventID"]
    if isinstance(eid, dict):
        eid = eid.get("#text")
    if not pat:
        cnt[eid] += 1
        continue
    if pat.search(line):
        d = e.get("EventData") or {}
        t = e["System"]["TimeCreated"]["#attributes"]["SystemTime"]
        extra = d.get("CommandLine") or d.get("TargetFilename") or d.get("TargetObject") or ""
        print(t, eid, d.get("Image", ""), extra, sep=" | ")

if not pat:
    for k, v in cnt.most_common():
        print(v, k)
```

| Parte | Qué hace | Por qué |
|---|---|---|
| `re.compile(sys.argv[2], re.I)` | Prepara una **expresión regular** con el segundo argumento; `re.I` ignora mayúsculas | Permite buscar `lsass\|mimikatz\|...` de una vez |
| `if len(sys.argv) > 2 else None` | Si no pasas segundo argumento, `pat` queda vacío | Así el mismo script tiene **dos modos**: recuento y búsqueda |
| `collections.Counter()` | Un diccionario que cuenta cuántas veces ve cada cosa | Para el recuento por ID |
| `for line in open(...)` | Lee el fichero línea a línea | Un evento por línea |
| `isinstance(eid, dict)` y `.get("#text")` | El `EventID` a veces es un número y a veces un diccionario con el número dentro de `#text` | Es un detalle del formato; sin esto el recuento sale mal |
| `pat.search(line)` | Busca el patrón en la **línea entera** (en bruto) | Por eso encuentra el patrón en cualquier campo, sin saber cuál |
| `d = e.get("EventData") or {}` | Si el evento no tiene `EventData`, usa un diccionario vacío | Evita errores |
| `d.get("CommandLine") or d.get("TargetFilename") or ...` | Coge **el primer valor que no esté vacío** | Distintos eventos guardan lo importante en campos distintos |
| `sep=" \| "` | Separa las columnas con una barra | Para leerlo mejor |

**Una pista que se vio en la práctica:** en la salida de la fase 4, los eventos ID 10 salieron **con las columnas vacías**. No era un fallo: el ID 10 no usa el campo `Image` sino `SourceImage` y `TargetImage`, y `q.py` solo miraba `Image`. Eso es lo que hizo falta `r.py` para ver el contenido completo. *Cuando una salida sale vacía, lo primero que hay que sospechar es que el script no mira el campo correcto, no que los datos no existan.*

### 20.4 `r.py`, línea a línea

```python
import sys, json
f = sys.argv[1]
eids = set(sys.argv[2].split(","))
for line in open(f):
    e = json.loads(line)["Event"]
    eid = e["System"]["EventID"]
    eid = eid.get("#text") if isinstance(eid, dict) else eid
    if str(eid) not in eids:
        continue
    t = e["System"]["TimeCreated"]["#attributes"]["SystemTime"]
    print(t, eid, json.dumps(e.get("EventData") or {}))
```

| Parte | Qué hace |
|---|---|
| `sys.argv[2].split(",")` | Convierte `"1,3,10"` en la lista `["1","3","10"]` |
| `set(...)` | La guarda como conjunto, para comprobar rápido si un ID está dentro |
| `str(eid) not in eids` + `continue` | Si el evento no es de los IDs pedidos, lo salta |
| `json.dumps(e.get("EventData") or {})` | Imprime **todos** los campos del evento, sin elegir ninguno |

La diferencia clave con `q.py`: **`q.py` busca; `r.py` enseña.** Con `q.py` localizas el candidato; con `r.py` lees el evento entero.

### 20.5 ¿Y sin Python?

Python fue una opción, no la única. Para tener claro que lo importante es la idea:

| Alternativa | Cuándo la usarías |
|---|---|
| Visor de eventos de Windows con filtro por ID y búsqueda de texto | Si tienes Windows y pocos eventos |
| Herramientas dedicadas (`chainsaw`, `hayabusa`, `EvtxECmd`) | Para analizar muchos logs de una vez con reglas ya hechas |
| `grep` sobre el JSON ya convertido | Para búsquedas rápidas (como la fase 4) |

### 20.6 Ejercicios: escríbelo tú (con pistas, sin soluciones)

Haz estos ejercicios **antes** de pasar al siguiente reto. Si te atascas, dime en qué línea y lo vemos; pero intenta primero.

1. **Contar por proceso.** Modifica `q.py` para que, solo con los eventos ID 3, cuente cuántas conexiones hizo cada `Image`. *Pista: usa el `Counter`, pero con otra clave.*
2. **Filtrar por tiempo.** Haz que `r.py` acepte una hora de inicio y otra de fin y solo imprima los eventos entre ambas. *Pista: las horas en formato ISO (`2024-09-13T13:40`) se pueden comparar como texto.*
3. **Ficheros en `C:\temp`.** Escribe un script (o un `grep`) que muestre los eventos ID 11 cuyo `TargetFilename` contenga `\temp\`. Mira si aparece el `OJBFQIVq.dmp`. *Pregunta:* ¿qué proceso lo creó?
4. **Hijos de PowerShell.** Lista todos los procesos (ID 1) cuyo `ParentImage` sea `powershell.exe`. *Pregunta:* ¿hay más cosas además del `rundll32`?
5. **Línea de tiempo.** Escribe un script nuevo, desde cero y sin mirar los anteriores, que imprima hora, ID y proceso de todos los eventos 1, 3 y 11 entre las 13:40 y las 13:50.

Los ejercicios 3 y 4 además sirven para **investigar lo que no sabemos** (sección 15).

---

## 21. Todos los comandos, juntos (copiar y pegar)

```bash
# 1) Triage por tamaño
ls -lS | head -25

# 2) Comprobar la librería y crear el conversor
python3 -c "import evtx; print(evtx.__file__)" 2>&1 || pip install evtx --break-system-packages
cat > ~/dump.py << 'EOF'
import sys, json
from evtx import PyEvtxParser

for rec in PyEvtxParser(sys.argv[1]).records_json():
    print(json.dumps(json.loads(rec["data"])))
EOF

# 3) Convertir Sysmon a JSON por líneas (usa ruta absoluta)
python3 ~/dump.py "RUTA/Microsoft-Windows-Sysmon%4Operational.evtx" > ~/sysmon.jsonl
wc -l ~/sysmon.jsonl

# 4) Mapa del log y búsqueda por patrones (q.py)
python3 ~/q.py ~/sysmon.jsonl
python3 ~/q.py ~/sysmon.jsonl 'lsass|mimikatz|procdump|comsvcs|Login Data|Local State|logins\.json|key4\.db|\.kdbx|webhook|pastebin|telegram' | head -50

# 5) Evento completo (r.py)
python3 ~/r.py ~/sysmon.jsonl 1 | grep -i comsvcs
python3 ~/q.py ~/sysmon.jsonl 'ProcessId": ?712' | head
python3 ~/r.py ~/sysmon.jsonl 10 | grep -i lsass
python3 ~/r.py ~/sysmon.jsonl 3
```
