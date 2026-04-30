# 🔐 Hacking Lab — Documentación de Hacking Ético

Repositorio personal de aprendizaje de hacking ético y ciberseguridad ofensiva. Contiene cheat sheets de herramientas, notas de máquinas resueltas y material del curso.

> ⚠️ **Aviso legal:** Todo el contenido de este repositorio está orientado exclusivamente a entornos controlados y laboratorios de práctica (HackMyVM, TryHackMe, VulnHub, DockerLabs). Ninguna técnica documentada aquí debe aplicarse sobre sistemas sin autorización explícita del propietario. El uso indebido de estas técnicas puede constituir un delito según el artículo 197 bis del Código Penal español.

---

## 🎯 Objetivo

Construir una base de conocimiento práctica y progresiva sobre:

- Reconocimiento y escaneo de redes
- Enumeración de servicios y aplicaciones web
- Explotación de vulnerabilidades conocidas
- Escalada de privilegios en Linux y Windows
- Análisis y manipulación de tráfico de red
- Auditoría WiFi y seguridad inalámbrica

---

## 📁 Estructura del repositorio

```
CTF-Writeups/
│
├── docs/                        # Cheat sheets por herramienta
│   ├── nmap.md                  # Escaneo de puertos y servicios
│   ├── gobuster.md              # Enumeración web (directorios, subdominios)
│   ├── hydra.md                 # Fuerza bruta de credenciales
│   ├── wpscan.md                # Enumeración y ataque WordPress
│   ├── metasploit.md            # Framework de explotación
│   ├── burpsuite.md             # Proxy HTTP y análisis web
│   ├── snmp.md                  # Enumeración SNMP
│   ├── john-hashcat.md          # Crackeo de hashes
│   ├── aircrack-ng.md           # Auditoría WiFi
│   ├── searchsploit.md          # Búsqueda de exploits locales
│   ├── xxe.md                   # Ataques XXE (XML External Entity)
│   └── ud0X-*.pdf               # Material teórico del curso
│
└── labs/                        # Writeups por máquina resuelta
    ├── friendly3.md             # HackMyVM — Friendly3
    ├── gameshell3.md            # HackMyVM — GameShell3
    ├── gameshell4-writeup.md    # HackMyVM — GameShell4
    ├── yuan112.md               # HackMyVM — Yuan112
    ├── yuan113.md               # HackMyVM — Yuan113
    ├── liar.md                  # HackMyVM — Liar
    ├── logan.md                 # HackMyVM — Logan
    ├── microchoft.md            # HackMyVM — Microchoft
    ├── gitdwn-hackmyvm.md       # HackMyVM — Gitdwn
    ├── fromytoy_writeup.md      # HackMyVM — Fromytoy
    ├── ms02423-hackmyvm.md      # HackMyVM — MS02423
    ├── Realsaga.md              # HackMyVM — Realsaga
    ├── poppins.md               # HackMyVM — Poppins
    ├── dc04-soupedecode-writeup.md  # HackMyVM — DC04 (Windows AD)
    ├── trust-dockerlabs.md      # DockerLabs — Trust
    ├── injection-dockerlabs.md  # DockerLabs — Injection
    └── wordpress-dockerlabs.md  # DockerLabs — WordPress
```

---

## 🛠️ Herramientas documentadas

| Herramienta    | Uso principal                             | Cheat sheet          |
|----------------|-------------------------------------------|----------------------|
| nmap           | Escaneo de puertos y detección de OS      | docs/nmap.md         |
| gobuster       | Fuerza bruta de directorios/vhosts        | docs/gobuster.md     |
| hydra          | Fuerza bruta de credenciales              | docs/hydra.md        |
| wpscan         | Enumeración y explotación WordPress       | docs/wpscan.md       |
| metasploit     | Explotación y post-explotación            | docs/metasploit.md   |
| burpsuite      | Intercepción y análisis de tráfico web    | docs/burpsuite.md    |
| snmp-check     | Enumeración vía protocolo SNMP            | docs/snmp.md         |
| john / hashcat | Crackeo de hashes                         | docs/john-hashcat.md |
| aircrack-ng    | Auditoría WiFi y captura de handshakes    | docs/aircrack-ng.md  |
| searchsploit   | Búsqueda de exploits en ExploitDB         | docs/searchsploit.md |

---

## 📋 Metodología de cada lab

Cada writeup en `labs/` sigue esta estructura:

1. **Reconocimiento** — descubrimiento de host y puertos abiertos
2. **Enumeración** — servicios, versiones, rutas web, usuarios
3. **Explotación** — obtención de acceso inicial
4. **Escalada de privilegios** — de usuario a root
5. **Flags** — captura de `user.txt` y `root.txt`
6. **Lecciones aprendidas** — técnicas y conceptos clave

---

## 📚 Material del curso

| Unidad | Tema                                  |
|--------|---------------------------------------|
| UD01   | Introducción al hacking ético         |
| UD02   | Protocolos y explotación web          |
| UD03   | Windows y Active Directory            |
| UD04   | WordPress y CMS                       |
| UD05   | Definición de objetivos de seguridad  |

---

## 🖥️ Plataformas utilizadas

- [HackMyVM](https://hackmyvm.eu)
- [TryHackMe](https://tryhackme.com)
- [VulnHub](https://vulnhub.com)
- [DockerLabs](https://dockerlabs.es)

---

## ⚙️ Entorno de trabajo

| Componente   | Detalle                                         |
|--------------|-------------------------------------------------|
| SO           | Kali Linux (rolling)                            |
| Arquitectura | Laboratorio local — VMs / contenedores Docker   |
| Hardware     | Acer Nitro V                                    |
| Antena WiFi  | Alfa AWUS036ACH (RTL8812AU) — monitor/inyección |
| VPN          | ProtonVPN                                       |
| Editor       | VS Code + Claude Code                           |

---

## 📊 Progreso

| Plataforma | Máquinas | Roots | Users |
|------------|----------|-------|-------|
| HackMyVM   | 14       | 14    | 14    |
| DockerLabs | 3        | 3     | 3     |

_Actualizado: Abril 2026_
