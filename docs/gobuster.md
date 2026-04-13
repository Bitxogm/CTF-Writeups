# 🔍 GOBUSTER - Directory & File Brute Forcer

> Herramienta para enumerar directorios, archivos y subdominios ocultos en servidores web.
> Esencial en la fase de reconocimiento web de cualquier CTF o pentest.

---

## ⚡ Flujo típico en un CTF / Lab

```
1. Escaneo rápido con common.txt    →  encontrar directorios básicos
2. Escaneo con extensiones          →  php, txt, html, bak
3. Wordlist grande si no hay nada   →  DirBuster medium/big
4. Buscar subdominios               →  modo dns
5. Revisar resultados               →  200 OK, 301 Redirect, 403 Forbidden
```

---

## 🔧 Comandos habituales

```bash
# Escaneo básico de directorios
gobuster dir -u http://192.168.0.10 -w /usr/share/wordlists/dirb/common.txt

# Con extensiones comunes
gobuster dir -u http://192.168.0.10 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak

# Wordlist grande (DirBuster)
gobuster dir -u http://192.168.0.10 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

# Guardar resultado
gobuster dir -u http://192.168.0.10 -w /usr/share/wordlists/dirb/common.txt -o resultado.txt

# Enumerar subdominios (modo DNS)
gobuster dns -d dominio.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Virtual hosts
gobuster vhost -u http://192.168.0.10 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Con autenticación básica
gobuster dir -u http://192.168.0.10 -w /usr/share/wordlists/dirb/common.txt -U admin -P password

# Ignorar certificado SSL
gobuster dir -u https://192.168.0.10 -w /usr/share/wordlists/dirb/common.txt -k
```

---

## 🚩 Flags principales

| Flag | Descripción |
|------|-------------|
| `-u` | URL objetivo |
| `-w` | Wordlist a usar |
| `-x` | Extensiones a buscar (php,txt,html) |
| `-o` | Guardar output en archivo |
| `-t` | Threads (por defecto 10) |
| `-k` | Ignorar errores SSL |
| `-b` | Códigos de estado a ignorar (por defecto 404) |
| `-s` | Códigos de estado a mostrar |
| `-U` | Usuario para autenticación básica |
| `-P` | Password para autenticación básica |
| `-a` | User-Agent personalizado |
| `-c` | Cookie personalizada |
| `--timeout` | Timeout por request |
| `-q` | Modo silencioso (solo resultados) |
| `-r` | Seguir redirects |

---

## 📊 Códigos de estado importantes

| Código | Significado | Acción |
|--------|-------------|--------|
| `200` | OK — existe y es accesible | ⭐ Investigar |
| `301/302` | Redirect | Seguir la redirección |
| `403` | Forbidden — existe pero sin acceso | Anotar, buscar bypass |
| `401` | Requiere autenticación | Buscar credenciales |
| `404` | No existe | Ignorar |
| `500` | Error del servidor | Puede ser interesante |

---

## 📁 Wordlists útiles en Kali (ordenadas por tamaño)

```bash
# Pequeñas (rápidas)
/usr/share/wordlists/dirb/small.txt
/usr/share/wordlists/dirb/common.txt              # ⭐ Empezar aquí

# Medianas
/usr/share/wordlists/dirb/big.txt
/usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt
/usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt

# Grandes (lentas)
/usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt

# Con extensiones específicas
/usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt
```

---

## 🧠 Estrategia en CTFs

```bash
# Paso 1 — rápido siempre
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -o paso1.txt

# Paso 2 — si no hay nada, ampliar
gobuster dir -u http://<IP> -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -o paso2.txt

# Paso 3 — buscar en subdirectorios encontrados
gobuster dir -u http://<IP>/directorio -w /usr/share/wordlists/dirb/common.txt -o paso3.txt

# Paso 4 — virtual hosts si el servidor lo soporta
gobuster vhost -u http://<IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## ⚠️ Notas importantes

- Siempre empezar con `common.txt` antes de wordlists grandes
- El código `403` significa que el archivo/directorio **existe** aunque no puedas acceder
- En CTFs, prestar atención a extensiones `.bak`, `.old`, `.txt` — suelen tener credenciales
- Si el servidor va lento, bajar threads con `-t 5`
- Guardar siempre con `-o` para no repetir escaneos

---

## 🔗 Referencias

- GitHub oficial: https://github.com/OJ/gobuster
- SecLists: https://github.com/danielmiessler/SecLists
