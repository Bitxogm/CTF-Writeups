Readme · MDCopy🔐 Hacking Lab — Documentación de Hacking Ético
Repositorio personal de aprendizaje de hacking ético y ciberseguridad ofensiva. Contiene cheat sheets de herramientas, notas de máquinas resueltas y material del curso.

⚠️ Aviso legal: Todo el contenido de este repositorio está orientado exclusivamente a entornos controlados y laboratorios de práctica (HackMyVM, TryHackMe, VulnHub, DockerLabs). Ninguna técnica documentada aquí debe aplicarse sobre sistemas sin autorización explícita del propietario. El uso indebido de estas técnicas puede constituir un delito según el artículo 197 bis del Código Penal español.


🎯 Objetivo
Construir una base de conocimiento práctica y progresiva sobre:

Reconocimiento y escaneo de redes
Enumeración de servicios y aplicaciones web
Explotación de vulnerabilidades conocidas
Escalada de privilegios en Linux y Windows
Análisis y manipulación de tráfico de red
Auditoría WiFi y seguridad inalámbrica


📁 Estructura del repositorio
hacking-lab/
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
    ├── gameshell4.md            # HackMyVM — GameShell4
    ├── yuan112.md               # HackMyVM — Yuan112
    ├── yuan113.md               # HackMyVM — Yuan113
    ├── liar.md                  # HackMyVM — Liar
    ├── logan.md                 # HackMyVM — Logan
    ├── microchoft.md            # HackMyVM — Microchoft
    ├── trust-dockerlabs.md      # DockerLabs — Trust
    ├── injection-dockerlabs.md  # DockerLabs — Injection
    ├── wordpress-dockerlabs.md  # DockerLabs — WordPress
    ├── asucar-dockerlabs.md     # DockerLabs — Asucar
    └── library-dockerlabs.md   # DockerLabs — Library

🛠️ Herramientas documentadas
HerramientaUso principalCheat sheetnmapEscaneo de puertos y detección de OSdocs/nmap.mdgobusterFuerza bruta de directorios/vhostsdocs/gobuster.mdhydraFuerza bruta de credencialesdocs/hydra.mdwpscanEnumeración y explotación WordPressdocs/wpscan.mdmetasploitExplotación y post-explotacióndocs/metasploit.mdburpsuiteIntercepción y análisis de tráfico webdocs/burpsuite.mdsnmp-checkEnumeración vía protocolo SNMPdocs/snmp.mdjohn / hashcatCrackeo de hashesdocs/john-hashcat.mdaircrack-ngAuditoría WiFi y captura de handshakesdocs/aircrack-ng.mdsearchsploitBúsqueda de exploits en ExploitDBdocs/searchsploit.md

📋 Metodología de cada lab
Cada writeup en labs/ sigue esta estructura:

Reconocimiento — descubrimiento de host y puertos abiertos
Enumeración — servicios, versiones, rutas web, usuarios
Explotación — obtención de acceso inicial
Escalada de privilegios — de usuario a root
Flags — captura de user.txt y root.txt
Lecciones aprendidas — técnicas y conceptos clave


📚 Material del curso
UnidadTemaUD01Introducción al hacking éticoUD02Protocolos y explotación webUD03Windows y Active DirectoryUD04WordPress y CMSUD05Definición de objetivos de seguridad

🖥️ Plataformas utilizadas

HackMyVM
TryHackMe
VulnHub
DockerLabs


⚙️ Entorno de trabajo

Sistema operativo: Kali Linux (rolling)
Arquitectura: laboratorio local con máquinas virtuales / contenedores Docker
Hardware: Acer Nitro V
Antena WiFi: Alfa AWUS036ACH (RTL8812AU) — modo monitor/inyección
VPN: ProtonVPN
Editor: VS Code + Claude Code


📊 Progreso
PlataformaRootsUsersHackMyVM77DockerLabs55
Actualizado: Abril 2026