# 🔒 Auditoría de Seguridad — talent.bitxodev.com

> **Tipo:** Verificación de seguridad propia  
> **Objetivo:** talent.bitxodev.com + infraestructura bitxodev.com  
> **Servidor:** Hetzner (detrás de Cloudflare)  
> **Fecha:** 2026-05-02  
> **Realizado por:** CSIRT Team / Claude

---

## 📊 Resumen Ejecutivo

| Severidad | Hallazgos |
|-----------|-----------|
| 🔴 CRÍTICO | 0 |
| 🟠 ALTO | 1 |
| 🟡 MEDIO | 2 |
| 🔵 INFO | 2 |
| ✅ OK | 8 |

---

## ✅ Controles Correctamente Configurados

| Control | Detalle |
|---------|---------|
| Cloudflare CDN/WAF | Activo — oculta IP real del servidor Hetzner |
| HTTPS + HTTP/2 | Activo en todos los endpoints |
| HSTS | `max-age=15552000; includeSubDomains; preload` ✓ |
| X-Frame-Options | `SAMEORIGIN` — protección clickjacking ✓ |
| X-Content-Type-Options | `nosniff` ✓ |
| X-XSS-Protection | `1; mode=block` ✓ |
| Referrer-Policy | `same-origin` ✓ |
| API keys en bundle JS | No expuestas — llamadas van al backend ✓ |
| `.git` / `.env` / `config.json` | No accesibles — SPA fallback correcto ✓ |

---

## 🟠 ALTO — JWT almacenado en localStorage

**Descripción:**  
El token de autenticación JWT se guarda en `localStorage` del navegador.

```javascript
localStorage.setItem("token", jwtToken)
```

**Riesgo:**  
Si existe una vulnerabilidad XSS en cualquier parte de la aplicación, un atacante puede robar el token con `document.cookie` o `localStorage.getItem("token")` y suplantar al usuario.

**Acción correctiva — Backend (`auth.controller.ts`):**

```typescript
// ANTES: devolver token en el body
res.json({ token: jwtToken });

// DESPUÉS: enviar como cookie HttpOnly
res.cookie('token', jwtToken, {
  httpOnly: true,       // JS no puede leerla
  secure: true,         // Solo HTTPS
  sameSite: 'strict',   // Protección CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000  // 7 días
});
res.json({ success: true });
```

**Acción correctiva — Backend (`server.ts` o `app.ts`):**

```typescript
import cookieParser from 'cookie-parser';
app.use(cookieParser());
```

**Acción correctiva — Middleware auth (`auth.middleware.ts`):**

```typescript
// ANTES
const token = req.headers.authorization?.split(' ')[1];

// DESPUÉS
const token = req.cookies.token;
```

**Acción correctiva — Frontend:**

```typescript
// Eliminar estas líneas:
localStorage.setItem("token", n)
localStorage.getItem("token")

// En todos los fetch/axios, añadir:
fetch('/api/...', {
  credentials: 'include'  // envía la cookie automáticamente
})
// o en axios:
axios.defaults.withCredentials = true;
```

**Instalación necesaria:**
```bash
cd backend && npm install cookie-parser @types/cookie-parser
```

---

## 🟡 MEDIO — Datos de CV en localStorage

**Descripción:**  
Los datos del CV y cartas de presentación (información personal sensible) se almacenan en `localStorage` del navegador.

```javascript
localStorage.setItem(CV_DATA, JSON.stringify(cvData))
localStorage.setItem(CV_HISTORY, JSON.stringify(history))
localStorage.setItem(COVER_LETTERS, JSON.stringify(coverLetters))
```

**Riesgo:**  
- Accesibles por cualquier JS de la página (riesgo XSS)
- Se pierden al limpiar caché del navegador
- No hay cifrado en reposo

**Acción correctiva:**  
Si los CVs ya se sincronizan con PostgreSQL via `/api`, el `localStorage` debe usarse solo como **caché temporal** (no como fuente de verdad). Verificar que:

1. Cada endpoint `/api/cv` valide que el `userId` del token coincide con el propietario del CV:

```typescript
// cv.controller.ts
const cv = await cvRepository.findById(cvId);
if (cv.userId !== req.user.id) {
  return res.status(403).json({ error: 'Acceso denegado' });
}
```

2. Los datos en `localStorage` se borren al hacer logout:

```typescript
// logout
localStorage.removeItem(CV_DATA);
localStorage.removeItem(CV_HISTORY);
localStorage.removeItem(COVER_LETTERS);
localStorage.removeItem('token');
```

---

## 🟡 MEDIO — Redirect HTTP en endpoint `/api`

**Descripción:**  
`GET /api` devuelve un `301` a `http://` (sin TLS) en lugar de `https://`.

```
HTTP/2 301
location: http://talent.bitxodev.com/api/
```

**Riesgo:** Bajo en producción (Cloudflare + HSTS lo corrige), pero indica una misconfiguration en nginx que podría afectar entornos sin Cloudflare.

**Acción correctiva — nginx:**

```nginx
# Asegurarse de que los redirects internos usen HTTPS
server {
    listen 443 ssl;
    ...
    location /api {
        return 301 https://$host/api/;
        # o usar proxy_pass sin trailing slash que cause redirect
    }
}
```

---

## 🔵 INFO — Versión de nginx expuesta

**Descripción:**  
La cabecera `Server` revela la versión exacta: `nginx/1.29.8`.

**Acción correctiva — nginx.conf:**

```nginx
http {
    server_tokens off;
}
```

---

## 🔵 INFO — Subdominios inactivos con error SSL 525

**Descripción:**  
3 subdominios con registros DNS activos pero SSL mal configurado en el origen:

- `academylogic.bitxodev.com` → Error 525
- `aidirectory.bitxodev.com` → Error 525
- `cvgenerator.bitxodev.com` → Error 525

**Riesgo:** Superficie de ataque innecesaria. Un subdominio inactivo puede ser objetivo de subdomain takeover.

**Acción correctiva:**  
Si no están en uso, eliminar los registros DNS en Cloudflare:

```
DNS → eliminar registros A/CNAME de:
  - academylogic.bitxodev.com
  - aidirectory.bitxodev.com
  - cvgenerator.bitxodev.com
```

---

## 🔒 Consideraciones GDPR — Datos de CVs

Los CVs contienen datos personales sensibles (nombre, dirección, teléfono, historial laboral). Obligaciones:

| Obligación | Estado | Acción |
|------------|--------|--------|
| Datos cifrados en BD | Verificar | Confirmar cifrado en PostgreSQL |
| Acceso solo al propietario | Verificar | Validar ownership en cada endpoint |
| Derecho de supresión | Implementar | Endpoint `DELETE /api/cv/:id` con borrado real |
| Política de privacidad | Recomendado | Añadir página de privacidad en la web |

---

## 📋 Plan de Acción Priorizado

| Prioridad | Acción | Dificultad |
|-----------|--------|------------|
| 1 | Migrar JWT de localStorage a cookie HttpOnly | Media |
| 2 | Validar ownership en endpoints de CV | Baja |
| 3 | Limpiar localStorage en logout | Baja |
| 4 | Eliminar subdominios DNS inactivos | Muy baja |
| 5 | `server_tokens off` en nginx | Muy baja |
| 6 | Corregir redirect HTTP→HTTPS en nginx | Baja |
