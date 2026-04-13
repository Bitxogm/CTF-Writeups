# Hacking Lab — Documentación de Hacking Ético

Repositorio personal de aprendizaje de hacking ético y ciberseguridad ofensiva.
Contiene cheat sheets de herramientas, notas de máquinas resueltas y material del curso.

> **Aviso legal:** Todo el contenido de este repositorio está orientado exclusivamente
> a entornos controlados y laboratorios de práctica (HackMyVM, TryHackMe, VulnHub,
> DockerLabs). Ninguna técnica documentada aquí debe aplicarse sobre sistemas sin
> autorización explícita del propietario.

---

## Objetivo

Construir una base de conocimiento práctica y progresiva sobre:

- Reconocimiento y escaneo de redes
- Enumeración de servicios y aplicaciones web
- Explotación de vulnerabilidades conocidas
- Escalada de privilegios en Linux y Windows
- Análisis y manipulación de tráfico de red

---

## Estructura del repositorio

```
hacking-lab/
│
├── docs/                        # Cheat sheets por herramienta
│   ├── nmap.md                  # Escaneo de puertos y servicios
│   ├── gobuster.md              # Enumeración web (directorios, subdominios)
│   ├── hydra.md                 # Fuerza bruta de credenciales
│   ├── metasploit.md            # Framework de explotación
│   ├── burpsuite.md             # Proxy HTTP y análisis web
│   ├── snmp.md                  # Enumeración SNMP
│   ├── xxe.md                   # Ataques XXE (XML External Entity)
│   └── ud0X-*.pdf               # Material teórico del curso
│
└── labs/                        # Notas por máquina resuelta
    ├── friendly3.md             # HackMyVM — Friendly3
    ├── gameshell3.md            # HackMyVM — GameShell3
    ├── yuan112.md               # HackMyVM — Yuan112
    ├── yuan113.md               # HackMyVM — Yuan113
    ├── liar.md                  # HackMyVM — Liar
    ├── logan.md                 # HackMyVM — Logan
    ├── microchoft.md            # HackMyVM — Microchoft
    ├── trust-dockerlabs.md      # DockerLabs — Trust
    ├── injection-dockerlabs.md  # DockerLabs — Injection
    └── wordpress-dockerlabs.md  # DockerLabs — Wordpress
```

---

## Herramientas documentadas

| Herramienta  | Uso principal                          | Cheat sheet              |
|--------------|----------------------------------------|--------------------------|
| nmap         | Escaneo de puertos y detección de OS   | [docs/nmap.md](docs/nmap.md) |
| gobuster     | Fuerza bruta de directorios/vhosts     | [docs/gobuster.md](docs/gobuster.md) |
| hydra        | Fuerza bruta de credenciales           | [docs/hydra.md](docs/hydra.md) |
| metasploit   | Explotación y post-explotación         | [docs/metasploit.md](docs/metasploit.md) |
| burpsuite    | Intercepción y análisis de tráfico web | [docs/burpsuite.md](docs/burpsuite.md) |
| snmp-check   | Enumeración vía protocolo SNMP         | [docs/snmp.md](docs/snmp.md) |

---

## Metodología de cada lab

Cada máquina en `labs/` sigue esta estructura:

1. **Reconocimiento** — descubrimiento de host y puertos abiertos
2. **Enumeración** — servicios, versiones, rutas web, usuarios
3. **Explotación** — obtención de acceso inicial
4. **Escalada de privilegios** — de usuario a root
5. **Flags** — captura de user.txt y root.txt

---

## Material del curso

| Unidad | Tema                                  |
|--------|---------------------------------------|
| UD01   | Introducción al hacking ético         |
| UD02   | Protocolos y explotación web          |
| UD03   | Windows y Active Directory            |
| UD04   | WordPress y CMS                       |

---

## Plataformas utilizadas

- [HackMyVM](https://hackmyvm.eu)
- [TryHackMe](https://tryhackme.com)
- [VulnHub](https://www.vulnhub.com)
- [DockerLabs](https://dockerlabs.es)

---

## Entorno de trabajo

- Sistema operativo: Kali Linux
- Arquitectura: laboratorio local con máquinas virtuales / contenedores Docker
