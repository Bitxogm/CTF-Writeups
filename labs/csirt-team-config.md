# Equipo CSIRT — Configuración y Registro de Acciones

> Documentación del equipo Claude CSIRT activo en `~/csirt/`, sus roles, configuración y acciones tomadas en sesiones anteriores.

---

## Configuración del equipo (`~/csirt/CLAUDE.md`)

El equipo opera bajo el perfil **CSIRT TEAM LEADER** con cuatro roles coordinados:

| Rol | Color | Función |
|-----|-------|---------|
| Líder | 🔵 | Coordinación, clasificación de severidad (CRITICAL/HIGH/MEDIUM), priorización de acciones |
| Analista | 🔴 | Ejecución técnica, TTPs MITRE ATT&CK, comandos Kali, análisis de IOCs |
| Forense | 🟡 | Orden de volatilidad, cadena de custodia, preservación de evidencias |
| Comunicaciones | 🟢 | Reporting legal/ejecutivo, cumplimiento GDPR (72h), NIS2, notificación CERTs |

**Prioridad de explotación:** RCE > Credenciales > SUID > Escalada de privilegios > Escape de contenedor

**Entornos:** Kali Linux, DockerLabs, HackMyVM

---

## Perfil del operador

- **Usuario:** ladybaba / ladybaba2000 en HackMyVM
- **Ranking:** Top 10 (posición #9, 143 pts a 2026-04-30)
- **Plataformas activas:** HackMyVM, DockerLabs, TryHackMe, HackTheBox
- **Capacidades:** recon, explotación web, esteganografía, escalada Linux, Active Directory, pivoting

---

## Feedback operacional registrado

1. **Comandos de uno en uno** — En fases críticas (escalada, explotación), dar comandos individualmente cuando el usuario lo pide. Tener varias terminales activas (shell, listener, servidor HTTP) genera confusión si se mezclan.

2. **Sin comandos multilinea con `\` en zsh** — El usuario usa zsh y los saltos de línea con `\` al pegar fallan frecuentemente. Dar siempre `curl` y comandos largos en una única línea.

---

## Registro de acciones por proyecto

### Pulse — HackMyVM (Intermediate) — COMPLETADA 2026-09-29

**Credenciales obtenidas:**
| Recurso | Usuario | Contraseña/Token |
|---------|---------|-----------------|
| SSH | alex | starlight (SHA-512 crackeado de users.db) |
| API /diagnostics | — | `pU1s3_d14gn0st1cs_k3y_98234` (git history) |
| Flask app | — | SECRET_KEY: `c798e1f02a45b891d2ef64098bc19a32` |

**Cadena de explotación:**
```
robots.txt → git-dumper → SSTI bypass blacklist → RCE www-data
→ SQLite users.db → hashcat starlight → SSH alex → user.txt
→ sudo SETENV PULSE_CONFIG → JSON malicioso → chmod SUID bash → root
```

**Flags:** user.txt `HMV{9b3f41c7e2a84d60b5e9f1a23c8e715d}` | root.txt `HMV{4d2c88f7b301e5a96c147289d038fa42}`

**Writeup completo:** [labs/pulse-hackmyvm.md](pulse-hackmyvm.md)

---

### SOC1 — HackMyVM — EN PROGRESO (última sesión: 2026-05-17)

**Target:** 192.168.0.147 (MAC: `08:00:27:42:8C:DF`, IP cambia en cada boot)

**Credenciales obtenidas:**
| Servicio | Usuario | Contraseña |
|----------|---------|-----------|
| Jenkins (8080) | test | test1234 (bcrypt crackeado con john) |
| Linux/Splunk | splunk | splunk123 (extraído de /etc/passwd.old) |
| Splunk web (8000) | splunk | splunk123 |

**Acceso conseguido:** Shell como `jenkins` via Groovy Script Console. `user.txt` leído.

**Sudo jenkins:**
```
(ALL) NOPASSWD: /opt/splunk/bin/splunk search *
(ALL) NOPASSWD: /opt/splunk/bin/splunk restart
```

**Ruta a root planificada:**
1. CVE-2023-46214 → shell como `splunk`
2. Como `splunk`: reemplazar `/opt/splunk/bin/splunk` con script malicioso
3. Desde `jenkins`: `sudo /opt/splunk/bin/splunk restart` → ejecuta como root

**Bloqueo activo:** El exploit CVE-2023-46214 sube el XSL OK pero el path en el parámetro `?xsl=` no es correcto. Splunk rechaza paths fuera de `$SPLUNK_HOME`. Próximo paso:
```bash
find /opt/splunk/ -name "shell.xsl" 2>/dev/null
find /opt/splunk/var/run/splunk/ -newer /tmp/cve.py 2>/dev/null | head -20
```

**Archivos en Kali persistentes:**
- `/tmp/splunk-rce/CVE-2023-46214.py` — exploit modificado
- `/tmp/shell.xsl` — XSL válido sin newlines, IP 192.168.0.145 port 5555

---

### Auditoría web — talent.bitxodev.com — AUTORIZADA

Pentest autorizado del servidor propio del usuario en Hetzner.  
Proyecto final de bootcamp. Metodología: recon → enumeración → explotación web → reporte.

**Informe generado:** [auditoria-talent.bitxodev.com](../auditoria-talent-bitxodev.md)

---

### Anaximandre — HackMyVM — PENDIENTE

Máquina WordPress. Vector probable: plugin vulnerable (RCE/SQLi) o credenciales débiles en wp-admin.  
Preparar: `wpscan`, enumeración de plugins/themes/usuarios desde el inicio.

**Writeup csirt:** `~/csirt/writeups/anaximandre.md`

---

## Estructura de archivos del equipo

```
~/csirt/
├── CLAUDE.md               — Configuración del equipo CSIRT
├── writeups/               — Writeups completos de máquinas resueltas
│   ├── pulse_hackymvm.md
│   ├── anaximandre.md
│   └── latestwasalie.md
└── labs/                   — Notas activas por máquina
    ├── aurora-hackmyvm.md
    ├── suidyrevenge-hackmyvm.md
    └── ... (resto de labs)
```
