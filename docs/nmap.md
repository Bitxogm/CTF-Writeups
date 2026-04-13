# 🗺️ NMAP - Network Mapper

> Herramienta de exploración de redes y auditoría de seguridad.  
> Descubre hosts activos, puertos abiertos, servicios, versiones y sistemas operativos.

---

## ⚡ Flujo típico en un CTF / Lab

```
1. Descubrir host       →  arp-scan / ping
2. Todos los puertos    →  -p- --min-rate 5000
3. Escaneo profundo     →  -sC -sV sobre puertos encontrados
4. Guardar resultados   →  -oN archivo
```

---

## 🔧 Comandos habituales

```bash
# Descubrimiento de red
sudo arp-scan -l

# Escaneo rápido de todos los puertos
sudo nmap -p- --min-rate 5000 -n -Pn <IP> -oN puertos

# Escaneo profundo (sobre los puertos encontrados)
sudo nmap -p 21,22,80 -sS -sC -sV -n -Pn <IP> -oN servicios

# OS + versiones + scripts (todo en uno)
sudo nmap -A <IP>

# Vulnerabilidades
sudo nmap --script=vuln <IP>

# FTP anónimo
sudo nmap --script=ftp-anon -p 21 <IP>

# UDP top puertos
sudo nmap -sU --top-ports 200 <IP>
```

---

## 🚩 Flags por categoría

### Puertos
| Flag | Descripción |
|------|-------------|
| `-p-` | Todos los puertos (1-65535) |
| `-p 22,80,443` | Puertos específicos |
| `-p 1-1000` | Rango de puertos |
| `-F` | Top 100 puertos (rápido) |
| `--top-ports 1000` | Top N puertos más comunes |
| `--exclude-ports 443` | Excluir puertos |

### Técnicas de escaneo
| Flag | Descripción |
|------|-------------|
| `-sS` | SYN scan — stealth, requiere root ⭐ |
| `-sT` | TCP connect — sin root |
| `-sU` | UDP scan |
| `-sV` | Detectar versión de servicios |
| `-sC` | Scripts por defecto (= --script=default) |
| `-sn` | Solo ping, sin escaneo de puertos |
| `-sO` | Escaneo de protocolo IP |

### Host Discovery
| Flag | Descripción |
|------|-------------|
| `-Pn` | Sin ping — trata el host como activo |
| `-n` | Sin resolución DNS (más rápido) |
| `-R` | Siempre resolver DNS |

### Velocidad
| Flag | Descripción |
|------|-------------|
| `-T0` | Paranoico (IDS evasion) |
| `-T1` | Sigiloso |
| `-T3` | Normal (por defecto) |
| `-T4` | Agresivo ⭐ |
| `-T5` | Insano |
| `--min-rate 5000` | Mínimo 5000 paquetes/seg |
| `--max-rate 1000` | Máximo paquetes/seg |

### OS y versiones
| Flag | Descripción |
|------|-------------|
| `-O` | Detectar sistema operativo |
| `-A` | Agresivo: OS + versión + scripts + traceroute |
| `--version-intensity 0-9` | Profundidad detección de versión |

### Output
| Flag | Descripción |
|------|-------------|
| `-oN archivo` | Formato normal (legible) |
| `-oX archivo` | Formato XML |
| `-oG archivo` | Formato grepable |
| `-oA basename` | Los tres formatos a la vez ⭐ |
| `-v` / `-vvv` | Verbosidad — ver puertos en tiempo real |

---

## 📜 Scripts NSE útiles

| Script | Descripción |
|--------|-------------|
| `--script=default` | Scripts básicos seguros |
| `--script=vuln` | Detección de vulnerabilidades |
| `--script=ftp-anon` | Comprobar acceso FTP anónimo |
| `--script=http-enum` | Enumerar directorios HTTP |
| `--script=http-title` | Ver título de páginas web |
| `--script=ssh-brute` | Fuerza bruta SSH |
| `--script=smb-vuln*` | Vulnerabilidades SMB (EternalBlue etc.) |
| `--script=banner` | Capturar banners de servicios |

---

## ⚠️ Notas importantes

- `-sS` **requiere sudo** — sin él usa `-sT` automáticamente
- Siempre guardar con `-oN` para no repetir escaneos
- `-vvv` muestra puertos en tiempo real sin esperar al final
- `--min-rate 5000` genera ruido — en entornos reales bajar velocidad
- En laboratorios locales (`192.168.x.x`) puedes usar `-T4` sin problema

---

## 🔗 Referencias

- Documentación oficial: https://nmap.org/docs.html
- NSE scripts: https://nmap.org/nsedoc/
- HackMyVM: https://hackmyvm.eu