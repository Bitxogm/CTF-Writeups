# Burp Suite

> Proxy de interceptación HTTP/S para análisis y ataque de aplicaciones web.
> Permite interceptar, modificar, repetir y automatizar peticiones.

---

## Flujo típico en un CTF / Lab

```
1. Configurar proxy         →  127.0.0.1:8080 en el navegador
2. Interceptar petición     →  Proxy > Intercept ON
3. Analizar / modificar     →  Forward o Send to Repeater
4. Repetir con cambios      →  Repeater (Ctrl+R)
5. Fuerza bruta params      →  Intruder (Ctrl+I)
6. Escanear vulns           →  Scanner (Pro) / plugins
```

---

## Configuración inicial

```
1. Burp Suite → Proxy → Options → Add/Edit listener → 127.0.0.1:8080
2. Navegador → Configuración → Proxy manual → 127.0.0.1:8080
3. Para HTTPS: instalar certificado CA de Burp
   - Navegar a http://burp con proxy activo
   - Descargar CA Certificate
   - Importar en el navegador (Authorities)
```

**FoxyProxy** (extensión Firefox recomendada) para cambiar proxy rápidamente.

---

## Herramientas principales

### Proxy
- **Intercept ON/OFF** — capturar peticiones en tiempo real
- **HTTP history** — ver todas las peticiones del historial
- **Forward** — enviar petición interceptada tal cual
- **Drop** — descartar petición
- **Action > Send to...** — enviar a Repeater, Intruder, etc.

### Repeater (Ctrl+R)
- Repetir y modificar peticiones manualmente
- Ver la respuesta del servidor en tiempo real
- Ideal para: probar payloads, bypass, explorar endpoints

### Intruder (Ctrl+I)
- Fuerza bruta / fuzzing automatizado
- Marcar posiciones con `§payload§`
- **Attack types:**
  - Sniper — una lista, una posición
  - Battering ram — misma lista en todas las posiciones
  - Pitchfork — listas separadas por posición
  - Cluster bomb — producto cartesiano de todas las listas

### Decoder
- Encode/decode: Base64, URL, HTML, Hex, etc.
- Útil para analizar tokens, cookies, parámetros ofuscados

### Comparer
- Comparar dos peticiones/respuestas lado a lado

---

## Atajos de teclado

| Atajo | Acción |
|-------|--------|
| `Ctrl+R` | Enviar a Repeater |
| `Ctrl+I` | Enviar a Intruder |
| `Ctrl+F` | Buscar en respuesta |
| `Ctrl+Z` | Deshacer en editor |
| `Ctrl+Space` | Autocompletar |
| `Ctrl+Shift+F` | Buscar en todo el proyecto |

---

## Intruder — ejemplo login brute force

```
POST /login HTTP/1.1
Host: 192.168.1.10

username=§admin§&password=§password§
```

- Positions: marcar `admin` y `password` como §payload§
- Payloads: cargar wordlists para cada posición
- Grep Match: añadir "Invalid" para identificar fallos rápido
- Ordenar por Length para encontrar respuesta distinta (éxito)

---

## Extensiones útiles (BApp Store)

| Extensión | Uso |
|-----------|-----|
| **Logger++** | Logging avanzado de peticiones |
| **Autorize** | Test de control de acceso / IDOR |
| **JWT Editor** | Manipulación de tokens JWT |
| **SQLiPy** | Integración con SQLMap |
| **Turbo Intruder** | Intruder de alta velocidad |
| **Active Scan++** | Más checks de seguridad |
| **Hackvertor** | Transformaciones de encoding avanzado |

---

## Notas importantes

- Burp Community limita la velocidad del Intruder — usar Turbo Intruder como alternativa
- Guardar el proyecto frecuentemente (Community no guarda automático)
- `Ctrl+Z` en Repeater para deshacer cambios en la petición
- Para APIs REST: cambiar `Content-Type: application/json` y adaptar payload
- Scope: definir en Target > Scope para evitar escanear fuera del objetivo

---

## Referencias

- Documentación: https://portswigger.net/burp/documentation
- Web Security Academy (labs gratis): https://portswigger.net/web-security
