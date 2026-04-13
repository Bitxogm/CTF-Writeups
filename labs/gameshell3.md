# 🎮 GameShell3 - HackMyVM (Beginner)

## 📋 Info

| Campo | Detalle |
|---|---|
| Máquina | GameShell3 |
| Plataforma | HackMyVM |
| Dificultad | Beginner |
| OS | Linux Debian |
| IP | 192.168.0.24 (variable) |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
sudo nmap -p- --min-rate 5000 -n -Pn 192.168.0.24
```

**Puertos abiertos:**
- 22/SSH
- 80/HTTP → Apache "Random Gate - Choose Your Door"
- 8001-8010/HTTP → ttyd (terminal web con buscaminas)

Escaneo de servicios:
```bash
sudo nmap -p 22,80,8001-8010 -sS -sC -sV -n -Pn 192.168.0.24
```

---

## 🌐 Enumeración Web

La web del puerto 80 muestra 10 puertas numeradas del 8001 al 8010. Al hacer clic en una puerta, si es la "ganadora" muestra un mensaje de éxito. El código JavaScript selecciona una puerta ganadora aleatoriamente.

Cada puerto del 8001 al 8010 tiene un servidor **ttyd** con un buscaminas en terminal. **Solo uno de los puertos tiene el buscaminas funcional** — el resto están congelados. Hay que probar cada puerto en el navegador para encontrar cuál responde al teclado.

En nuestro caso el puerto funcional fue el **8002**.

---

## 🎯 Obtención de Credenciales — Buscaminas

Al acceder al puerto funcional en el navegador:
```
http://192.168.0.24:8002
```

Aparece un buscaminas jugable en terminal. Controles:
- `WASD` → mover cursor
- `Space` → descubrir celda
- `F` → marcar mina
- `Q` → salir

**Al ganar el buscaminas** aparece el mensaje:
```
Congrats skr, you Win!
You swept all mines in 54 seconds.
Your pass: skrampy1
```

**Credenciales SSH:** `skr:skrampy1`

---

## 🔑 Acceso Inicial — SSH

```bash
ssh skr@192.168.0.24
# Password: skrampy1
```

Una vez dentro, deshabilitar el auto-logout:
```bash
export TMOUT=0
```

**Flag user:**
```bash
cat ~/user.txt
# flag{user-a2a53d2efdda06bc16093ad7b3551709}
```

---

## 📁 Enumeración del Sistema

```bash
ls -la /var/backups/
```

Archivo interesante: `hidden.img` (100MB — imagen ext4)

```bash
file /var/backups/hidden.img
# Linux rev 1.0 ext4 filesystem data
```

Buscar archivos dentro sin montar:
```bash
strings /var/backups/hidden.img | grep -i "secret\|wav\|music"
# secretmusic
# WAVEfmt
```

---

## 🎵 Extracción del Audio

Encontrar el offset del archivo WAV dentro de la imagen:
```bash
grep -oba "RIFF" /var/backups/hidden.img | head -5
# 8653824:RIFF
```

Extraer el WAV con dd:
```bash
dd if=/var/backups/hidden.img of=/tmp/secretmusic.wav bs=1 skip=8653824 count=27245
```

Transferir a la máquina atacante:
```bash
# En la víctima
cd /tmp && python3 -m http.server 8080

# En Kali
curl http://192.168.0.24:8080/secretmusic.wav -o /tmp/secretmusic.wav
file /tmp/secretmusic.wav
# RIFF (little-endian) data, WAVE audio, Microsoft PCM, 8 bit, mono 8000 Hz
```

---

## 📞 Decodificación DTMF

El archivo WAV contiene tonos DTMF (tonos de teclado telefónico). Subir el archivo a:

```
https://dtmf.netlify.app/
```

**Resultado decodificado:**
```
*#*#660930334#*#*
```

---

## 🔴 Escalada a Root

```bash
su - root
# Password: *#*#660930334#*#*
```

**Flag root:**
```bash
cat /root/root.txt
# flag{root-f0cc428ad5cb90aebdfc7aa4e778b2cc}
```

---

## 🗺️ Cadena de Ataque

1. **Nmap** → descubre puertos 80 y 8001-8010 con ttyd
2. **Web** → Random Gate indica explorar los 10 puertos
3. **Puerto 8002** → buscaminas funcional → ganar → credenciales `skr:skrampy1`
4. **SSH** → acceso como `skr` + `export TMOUT=0`
5. **Flag user** → `cat ~/user.txt`
6. **/var/backups/hidden.img** → imagen ext4 con archivo WAV oculto
7. **grep + dd** → extraer WAV sin montar la imagen
8. **DTMF decoder** → decodificar tonos → contraseña root `*#*#660930334#*#*`
9. **su - root** → ROOT
10. **Flag root** → `cat /root/root.txt`

---

## 🛠️ Herramientas Usadas

- `nmap` — reconocimiento de puertos
- `arp-scan` — descubrimiento de IP
- Navegador — jugar buscaminas en ttyd
- `strings` — buscar contenido en imagen
- `grep -oba` — encontrar offset binario
- `dd` — extraer archivo de imagen
- `python3 -m http.server` — transferir archivos
- https://dtmf.netlify.app/ — decodificar tonos DTMF

---

## 📝 Técnicas Aprendidas

- **ttyd** — terminal web sin autenticación accesible vía navegador
- **Imagen ext4** — explorar sin montar con `strings` y `grep -oba`
- **dd con offset** — extraer archivos binarios de imágenes
- **DTMF** — tonos telefónicos usados como contraseña (esteganografía de audio)
- **debugfs** — alternativa para explorar imágenes ext4 (si está disponible)
