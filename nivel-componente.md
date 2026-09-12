# Nivel componente

Resumen de backend, frontend público y frontend administrativo. No incluye rutas/endpoints, contratos de API, módulos internos ni cobertura de pruebas — ver `AGENTS.md` de cada repo para eso.

## Backend (`restobite-backend`, NestJS)

- **Propósito**: API central multi-tenant para gestión de restaurantes (menú, categorías, ítems, personalizaciones, promociones, sucursales), registro/onboarding de tenants y assets.
- **Estado**: activo.
- **Con quién habla**: recibe llamadas del frontend administrativo (JWT, desde el navegador) y del frontend público (HMAC, server-side). Habla hacia afuera con PostgreSQL (Prisma), S3, SES y reCAPTCHA Enterprise. Emite logs hacia stdout, archivo y/o Loki según configuración.
- **Autenticación**: tres mecanismos conviven — JWT (Bearer, para el frontend administrativo, expira a los 7 días sin refresh), credenciales HMAC por header (para llamadas server-to-server, usadas por el frontend público) y un login local (`tenantCode:usuario` + password + reCAPTCHA) que emite el JWT. Un guard global exige JWT o HMAC en toda ruta salvo las marcadas explícitamente como públicas.
- **Dónde corre / despliegue**: contenedor Docker en el VPS, detrás de Nginx (puerto interno 4000). Actualización mediante contenedor auxiliar + swap + reload de Nginx (`scripts/create-updated-aux-container.sh`, `scripts/update-docker-container.sh`).

## Frontend público (`restobite-public`, Next.js 15)

- **Propósito**: sitio público de Restobite (landing) y páginas de restaurante por tenant, resueltas por path (`/-/{tenantCode}`), no por subdominio.
- **Estado**: activo.
- **Con quién habla**: llama al backend únicamente desde el servidor (SSR / server actions de Next.js) — el navegador nunca llama al backend directamente. Sirve imágenes desde `assets.restobite.com` (CloudFront/S3).
- **Autenticación**: no hay login de usuario final. La comunicación servidor→backend se autentica con firma HMAC (credencial + timestamp + firma por header), no con JWT. No se encontró mecanismo de resolución de tenant por subdominio en este repo ni en la configuración de Nginx relevada.
- **Dónde corre / despliegue**: contenedor Docker en el VPS, detrás de Nginx (puerto interno 3000). No se encontró script de swap/blue-green equivalente al del backend para este repo — mecanismo de actualización en producción `desconocido`.

## Frontend administrativo (`restobite-admin`, React 18 + Vite)

- **Propósito**: panel de administración de un tenant (menú, info del negocio, promociones, QR/link, personalización de tema).
- **Estado**: activo.
- **Con quién habla**: llama al backend directo desde el navegador. Llama también, directo desde el navegador, a la API de Google Places (autocompletado de direcciones) — sin pasar por el backend.
- **Autenticación**: login con `tenantCode` + usuario/contraseña + reCAPTCHA v3 contra el backend, que devuelve un JWT. El JWT se guarda en `localStorage` del navegador y se envía como Bearer en cada request. No hay lógica de refresh: al expirar, se fuerza nuevo login.
- **Dónde corre / despliegue**: build estático servido por S3 + CloudFront (no corre en el VPS). Deploy mediante `scripts/deploy.sh`, que sincroniza el build (`aws s3 sync`) directo al bucket S3 — no hay swap de contenedor ni reload de Nginx involucrados, a diferencia del backend. Qué proceso dispara este script (Jenkins u otro) es `desconocido` (ver [cicd.md](infraestructura/cicd.md)).
