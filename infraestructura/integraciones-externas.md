# Integraciones externas

## AWS S3

- **Para qué se usa**: almacenamiento de assets (imágenes) subidos desde el frontend administrativo; también hospeda el sitio estático del frontend administrativo (`restobite-admin-app-prod`), los assets públicos servidos vía `assets.restobite.com`, el buzón de correo entrante de SES, y el bucket de redirect de `restobite.com`.
- **Cómo se usa (assets de aplicación)**: el backend (`S3AssetFileRepository`, `@aws-sdk/client-s3` + `s3-request-presigner`) genera URLs prefirmadas de subida (expiración 1 hora, key `tenants/{tenantId}/{filename}`) y de borrado; el navegador sube el archivo directo a S3 con esa URL, sin pasar el archivo por el backend.
- **Qué pasa si falla**: `desconocido` (sin fallback documentado en el código relevado).

## AWS SES

- **Para qué se usa**: envío de emails desde el backend (`SesEmailService`, `@aws-sdk/client-ses`), en particular el correo de bienvenida del flujo de registro de tenant (plantillas HTML en `restobite-backend/src/domains/tenant/email-templates/`).
- **Qué pasa si falla**: `desconocido`.

## AWS CloudFront

- **Para qué se usa**: CDN/edge delante de S3 (admin app, assets públicos, redirect de raíz) y delante del VPS (API y frontend público), con políticas de cache/origin-request propias por distribución.
- **Mecanismo de seguridad de origen**: las distribuciones que apuntan al VPS envían un header custom `X-Origin-Secret` que Nginx valida antes de hacer `proxy_pass` (ver [nginx.md](nginx.md)) — evita que el VPS reciba tráfico que no pasó por CloudFront.
- **Qué pasa si falla**: `desconocido`.

## AWS IAM

- **Para qué se usa**: un usuario IAM (`jenkins`) con policies inline de S3 (subida/borrado sobre el bucket admin-app, listado de buckets) y un access key asociado — permisos usados presumiblemente por el proceso de deploy del frontend administrativo (`aws s3 sync`, ver [nivel-componente.md](../nivel-componente.md)). Dónde se almacenan y usan esas credenciales dentro de Jenkins: `desconocido` (ver [cicd.md](cicd.md)).

## GCP Cloud SQL

- **Para qué se usa**: base de datos productiva PostgreSQL del backend (ver [persistencia.md](persistencia.md)).
- **Qué pasa si falla**: `desconocido` (sin estrategia de alta disponibilidad ni failover documentada en el código/Terraform relevado).

## Google reCAPTCHA Enterprise

- **Para qué se usa**: verificación anti-bot en dos flujos — login del frontend administrativo (`admin_login`) y registro de "tester" en la landing del frontend público (`public_tester_registration`).
- **Cómo se usa**: el backend (`RecaptchaService`, `@google-cloud/recaptcha-enterprise`) crea una assessment con el token recibido del frontend, valida `tokenProperties.valid`, que la acción coincida y que el score sea ≥ 0.5. Ambos frontends ejecutan el challenge en el navegador (`react-google-recaptcha-v3`) y envían el token al backend.
- **Bypass conocido**: en entornos `development`/`test`, un token literal `'bypass'` salta la verificación.
- **Qué pasa si falla**: la request es rechazada por el guard de reCAPTCHA (no se completa login ni registro).

## Google Places API

- **Para qué se usa**: autocompletado de direcciones en el formulario de restaurante del frontend administrativo (API "Places" nueva: `places:autocomplete` y `places/{placeId}`, con header `X-Goog-Api-Key`).
- **Dónde se llama**: directo desde el navegador (frontend administrativo), no desde el backend. No se encontró uso de Google Places en el backend ni en el frontend público.
- **Nota**: la misma API key (`PUBLIC_GOOGLE_API_KEY`) también se usa para Google Fonts en el mismo frontend, sin relación con Places. Existe una variable `PUBLIC_LOCATION_IQ_API_KEY` declarada en el frontend administrativo sin ningún uso encontrado en el código (variable no consumida). El renderizado del mapa en sí usa Leaflet + tiles de OpenStreetMap, no Google Maps.
- **Qué pasa si falla**: `desconocido`.
