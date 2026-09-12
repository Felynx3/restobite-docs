# Mapa de componentes

Vista de contenedores (nivel C4 "Container"). Sin rutas ni contratos — solo bloques y relaciones de tráfico/dependencia.

```mermaid
C4Container
    Person(usuarioPublico, "Usuario final", "Navega el sitio público de un restaurante")
    Person(usuarioAdmin, "Usuario administrador", "Gestiona el menú/negocio de un tenant")

    System_Boundary(vps, "VPS (Docker + Nginx)") {
        Container(nginx, "Nginx", "Docker", "Termina TLS, enruta por hostname")
        Container(backendC, "Backend", "NestJS, Docker", "API central")
        Container(publicC, "Frontend público", "Next.js, Docker", "Landing + páginas por tenant")
        Container(jenkins, "Jenkins", "Docker", "Build y deploy")
        Container(loki, "Loki", "Docker", "Almacena logs")
        Container(promtail, "Promtail", "Docker", "Recolecta logs de Nginx")
        Container(grafana, "Grafana", "Docker", "Consulta de logs")
    }

    Container(adminC, "Frontend administrativo", "React/Vite SPA, S3+CloudFront", "Panel de gestión")
    ContainerDb(db, "PostgreSQL", "GCP Cloud SQL, vía Prisma", "Persistencia principal, multitenant")

    System_Boundary(aws, "AWS") {
        Container(s3, "S3", "Buckets", "Assets, sitio admin, redirect, emails SES")
        Container(ses, "SES", "Email", "Envío de correo")
        Container(cloudfront, "CloudFront", "CDN", "Edge delante de S3 y del VPS")
        Container(route53, "Route 53", "DNS", "Dominios de restobite.com")
        Container(iam, "IAM", "Permisos", "Usuario jenkins para deploys")
    }

    System_Boundary(gcp, "GCP") {
        Container(recaptcha, "reCAPTCHA Enterprise", "API", "Verificación anti-bot")
    }

    System_Ext(googlePlaces, "Google Places API", "Externo a Restobite, sin relación con el proyecto GCP propio")

    Rel(usuarioPublico, cloudfront, "HTTPS")
    Rel(cloudfront, nginx, "origin, header X-Origin-Secret")
    Rel(nginx, publicC, "proxy_pass")
    Rel(publicC, backendC, "HTTPS + firma HMAC (server-side)")

    Rel(usuarioAdmin, cloudfront, "HTTPS (carga la SPA)")
    Rel(cloudfront, s3, "origin (sitio estático admin)")
    Rel(adminC, backendC, "HTTPS + JWT Bearer, desde el navegador, vía CloudFront/api.restobite.com")
    Rel(adminC, recaptcha, "reCAPTCHA v3 (browser)")
    Rel(adminC, googlePlaces, "Autocompletado de direcciones (directo, browser)")

    Rel(backendC, db, "Prisma")
    Rel(backendC, s3, "Assets: URL prefirmada de subida/lectura")
    Rel(backendC, ses, "Envío de emails")
    Rel(backendC, recaptcha, "Verificación de tokens")
    Rel(backendC, loki, "Logs JSON (winston-loki)")
    Rel(publicC, loki, "Logs JSON (winston-loki)")

    Rel(nginx, promtail, "Archivos de log (host compartido)")
    Rel(promtail, loki, "Push")
    Rel(grafana, loki, "Consulta")
    Rel(nginx, jenkins, "proxy_pass ci.restobite.com")
    Rel(nginx, grafana, "proxy_pass logs.restobite.com")
    Rel(iam, s3, "Credenciales de deploy (admin-app bucket)")
    Rel(route53, cloudfront, "Alias")
    Rel(route53, nginx, "A record (origin-*, ci, logs → IP del VPS)")
```

`Google Places API` se muestra como sistema externo (no es un componente de Restobite): representa la llamada directa del navegador (Frontend administrativo) a esa API, sin pasar por el backend.

## Componentes

- **Frontend público (Next.js)** — sirve el sitio por tenant (ruta `/-/{tenantCode}`) y la landing de Restobite; llama al backend solo desde el servidor (SSR/server actions), nunca desde el navegador.
- **Frontend administrativo (React/Vite)** — panel de gestión; SPA estática servida por S3+CloudFront; llama al backend directo desde el navegador y requiere login.
- **Backend (NestJS)** — API central; habla con PostgreSQL (vía Prisma), S3, SES, reCAPTCHA Enterprise, y emite logs. No se encontró integración con Google Places en el backend (ver [integraciones-externas.md](infraestructura/integraciones-externas.md)).
- **Base de datos (PostgreSQL / Prisma / multitenancy)** — persistencia principal, alojada en GCP Cloud SQL.
- **VPS (Docker + Nginx)** — corre efectivamente el backend, el frontend público, Jenkins y el stack de logs; Nginx termina TLS y enruta tráfico por hostname.
- **AWS (S3, SES, CloudFront, Route 53, IAM)** — assets y sitio admin (S3), correo (SES), CDN/edge (CloudFront), DNS (Route 53), credenciales de deploy (IAM).
- **GCP (Cloud SQL, reCAPTCHA Enterprise)** — base de datos productiva y verificación anti-bot. No se encontró uso de GCP como base paralela/alternativa a AWS: es la única base de datos relacional del sistema.
- **Observabilidad (Winston → Loki, y Nginx → Promtail → Loki → Grafana)** — dos rutas de ingesta distintas hacia el mismo Loki.
- **CI/CD (Jenkins + VPS)** — build y deploy; gran parte de su configuración vive fuera de Git (ver [cicd.md](infraestructura/cicd.md)).
