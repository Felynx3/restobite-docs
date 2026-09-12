# Flujos end-to-end

Cada flujo como secuencia de componentes del [mapa](mapa-componentes.md), no como secuencia de funciones o endpoints.

## Navegación pública por tenant

Usuario final → CloudFront → Nginx (VPS) → Frontend público (Next.js, resuelve el tenant por path `/-/{tenantCode}`) → Backend (llamada server-side firmada con HMAC) → PostgreSQL (Cloud SQL, consulta con `tenantId` forzado por Prisma) → Backend → Frontend público (renderiza) → Usuario final. Imágenes del restaurante se sirven aparte, directo desde `assets.restobite.com` (CloudFront → S3), sin pasar por el backend en cada carga.

## Login y edición administrativa

Usuario administrador → Frontend administrativo (SPA en el navegador) → reCAPTCHA Enterprise (challenge en el navegador) → Backend (`login local`: valida `tenantCode:usuario` + contraseña + token de reCAPTCHA) → Backend emite JWT → Frontend administrativo guarda el JWT en `localStorage`. Para cada edición posterior: Frontend administrativo (con `Authorization: Bearer <JWT>`) → Backend (valida JWT, fija el tenant en el contexto de la request) → PostgreSQL (Cloud SQL, escritura con `tenantId` forzado).

## Carga de assets

Frontend administrativo (navegador, JWT) → Backend (solicita subida de asset) → Backend genera URL prefirmada de S3 y responde con `{uploadUrl, assetId, publicUrl}` → Frontend administrativo sube el archivo directo a S3 con esa URL (sin pasar por el Backend) → Frontend administrativo notifica al Backend que la subida terminó → Backend confirma el asset. El `publicUrl` resultante (bajo `assets.restobite.com`, CloudFront → S3) queda disponible para el frontend administrativo y el frontend público.

## Despliegue (commit → Jenkins → VPS)

Desarrollador hace commit/push a uno de los repos de aplicación → Jenkins (contenedor en el VPS; disparo y configuración internos son incógnita, ver [cicd.md](infraestructura/cicd.md)) → según el repo:
- **Backend**: build de imagen → contenedor auxiliar en paralelo al productivo → verificación → Nginx (VPS) repunta su config al nuevo contenedor y recarga → se elimina el contenedor reemplazado.
- **Frontend público**: build de imagen Docker → mecanismo de actualización del contenedor productivo en el VPS: `desconocido` (no se encontró script de swap).
- **Frontend administrativo**: build estático → sincronizado directo a un bucket S3 (`aws s3 sync`) → servido por CloudFront; este camino no pasa por el VPS ni por Nginx.

## Envío y consulta de logs

Dos caminos en paralelo hacia Loki:
- **Directo**: Backend y Frontend público (si tienen `LOG_LOKI_URL` configurada) → Loki (push directo vía `winston-loki`, con `requestId` incluido en el JSON).
- **Vía Nginx**: Nginx (VPS) escribe accesos/errores del frontend público a archivo en `/var/data/nginx/logs` → Promtail lee esos archivos (sin parsear `requestId`) → Loki (push).

En ambos casos, consulta final: Usuario (operador) → Nginx (`logs.restobite.com`) → Grafana → Loki.
