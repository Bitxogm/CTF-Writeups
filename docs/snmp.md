# 📡 SNMP - Simple Network Management Protocol

> Protocolo de gestión de red que corre en UDP/161.  
> Permite monitorizar y gestionar dispositivos en red.  
> Mal configurado, puede exponer información crítica del sistema.

---

## ⚡ Flujo típico en un CTF / Lab

```
1. Escaneo UDP             →  nmap -sU -p161
2. Probar community string →  snmpwalk -v 2c -c public <IP>
3. Analizar output         →  buscar credenciales, procesos, usuarios
4. Explotar info obtenida  →  SSH, login con credenciales encontradas
```

---

## ⚠️ Por qué es importante escanear UDP

La mayoría de herramientas escanean **solo TCP** por defecto.
SNMP corre en **UDP/161** y puede quedar invisible si no se escanea UDP.

```bash
# Escaneo TCP normal — NO detecta SNMP
nmap 192.168.0.10

# Escaneo UDP — detecta SNMP ⭐
sudo nmap -sU -p161 192.168.0.10

# Escaneo UDP completo (lento)
sudo nmap -sU -p1-200 192.168.0.10
```

---

## 🔧 Comandos habituales

```bash
# Dump completo del MIB tree
snmpwalk -v 2c -c public <IP> > snmp_output.txt

# Buscar procesos corriendo (credenciales en CLI)
cat snmp_output.txt | grep "service"
cat snmp_output.txt | grep "password\|pass\|pwd\|user\|login"

# Buscar info del sistema
cat snmp_output.txt | grep "sysDescr\|sysName\|sysContact"

# Query directo a un OID específico
snmpget -v 2c -c public <IP> 1.3.6.1.2.1.1.1.0

# Probar diferentes community strings
snmpwalk -v 2c -c private <IP>
snmpwalk -v 2c -c manager <IP>
snmpwalk -v 1 -c public <IP>
```

---

## 🚩 Flags de snmpwalk

| Flag | Descripción |
|------|-------------|
| `-v 1` | SNMP versión 1 |
| `-v 2c` | SNMP versión 2c ⭐ más común |
| `-v 3` | SNMP versión 3 (con autenticación) |
| `-c public` | Community string (por defecto "public") |
| `-c private` | Community string alternativa |
| `-O e` | Output sin traducción de OIDs |

---

## 🔍 OIDs útiles para buscar

| OID | Descripción |
|-----|-------------|
| `1.3.6.1.2.1.1` | Info del sistema (sysDescr, sysName) |
| `1.3.6.1.2.1.25.4.2` | Procesos corriendo ⭐ |
| `1.3.6.1.2.1.25.6.3` | Software instalado |
| `1.3.6.1.2.1.6.13` | Conexiones TCP activas |
| `1.3.6.1.4.1.77.1.2.25` | Usuarios del sistema (Windows) |

---

## 💡 Truco clave — credenciales en procesos

Los administradores a veces pasan credenciales como argumentos en CLI:

```bash
# Esto aparece en SNMP como proceso corriendo
service --user admin --password secreto123 --host localhost

# Buscar en el output de snmpwalk
cat snmp_output.txt | grep "password\|--pass\|-p "
```

**Esto es un fallo crítico de seguridad** — cualquier proceso o servicio con acceso a `/proc` puede ver las credenciales.

---

## 🛡️ Community strings comunes a probar

```
public      ← por defecto, el más común
private     ← segundo más común
manager
community
admin
internal
```

---

## 📁 Herramientas relacionadas

```bash
# snmp-check — más legible que snmpwalk
snmp-check <IP>
snmp-check -c public <IP>

# onesixtyone — brute force de community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>

# snmpbulkwalk — más rápido que snmpwalk
snmpbulkwalk -v 2c -c public <IP>
```

---

## ⚠️ Notas importantes

- **Siempre escanear UDP** en CTFs — SNMP, DNS (53), TFTP (69), NTP (123)
- UDP scans son lentos — usar `--min-rate` con precaución en UDP
- Community string `public` sin cambiar = mala configuración
- SNMPv1 y v2c no tienen cifrado — credenciales en texto plano
- SNMPv3 es la versión segura con autenticación y cifrado

---

## 🔗 Referencias

- MIB Browser online: https://www.oid-info.com
- SecLists SNMP wordlists: `/usr/share/seclists/Discovery/SNMP/`
