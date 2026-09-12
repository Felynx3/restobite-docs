# Observabilidad

Componentes: Winston (en `restobite-backend` y `restobite-public`), Loki, Promtail y Grafana (repo `restobite-logs`), Nginx (repo `restobite-nginx`) como fuente adicional de logs.

## Formato de logs (backend)

JSON, generado por Winston (`restobite-backend/src/log/service/winston/`). Pipeline de formato: redacción de campos sensibles (`password`, `currentPassword`, `newPassword`, `token`, `recaptchaToken` → `[REDACTED]`), agregado de timestamp ISO, y serialización final con forma `{ requestId, timestamp, message, ...datos, level }`. El `requestId` se inyecta desde el contexto de request (CLS) en cada línea de log, permitiendo correlacionar todas las líneas de una misma request. El backend también loguea cada request/response HTTP (body y headers) vía middleware global.

## Destinos de logs del backend / frontend público

Tres transportes posibles, cada uno condicionado a una variable de entorno:
- `LOG_CONSOLE_ENABLED=true` → stdout.
- `LOG_FILE_PATH` seteada → archivo.
- `LOG_LOKI_URL` seteada → transporte `winston-loki`, directo a Loki, con labels `{ source: 'restobite-backend', env }` (o `restobite-public` según el emisor).

`restobite-public` declara las mismas variables (`LOG_CONSOLE_ENABLED`, `LOG_LOKI_URL`) en su `.env.example`, es decir también puede empujar logs directo a Loki.

## Rutas de ingestión a Loki

Hay **dos rutas independientes** hacia el mismo Loki:

1. **Directa**: backend y frontend público, si tienen `LOG_LOKI_URL` configurada, empujan sus logs JSON (con `requestId`) directo a Loki vía `winston-loki`.
2. **Vía Promtail**: Promtail (`restobite-logs/promtail/promtail-config.yaml`) solo scrapea dos archivos de log de **Nginx** — `restobite_public.log` (nivel `info`) y `restobite_public_error.log` (nivel `error`) — desde el path compartido `/var/data/nginx/logs`. Cada uno tiene labels estáticas (`env: production`, `level`, `source: restobite-nginx`). No hay `pipeline_stages` configurado: aunque Nginx escribe estos logs en formato `loki_json`, Promtail no los parsea como JSON ni extrae `requestId` — se envían como texto plano con esas labels fijas.

Consecuencia: los logs de Nginx en Loki no quedan correlacionados por `requestId` con los del backend, aunque el dato exista en el log de Nginx.

## Retención y almacenamiento en Loki

`restobite-logs/loki/loki-config.yaml`: `auth_enabled: false`, puerto `3100`, `retention_period: 168h` (7 días). No hay bloque `schema_config` ni `storage_config` explícito — Loki corre en modo single-binary con sus defaults; la persistencia es un volumen Docker nombrado (`restobite-logs-loki:/loki`), es decir almacenamiento en filesystem local del VPS, no S3 ni otro backend remoto.

## Grafana

Se accede vía `logs.restobite.com` (Nginx → contenedor `restobite-logs-grafana`). No hay dashboards ni datasources provisionados como código en el repositorio — el propio `AGENTS.md` de `restobite-logs` indica que `grafana/data/` (que contendría esa configuración en runtime) se mantiene sin commitear. Contenido actual de dashboards/datasources y consultas típicas usadas: `desconocido`.

## Límites conocidos

- Los logs de Nginx no llegan a Loki con `requestId`, pese a que el formato de origen (`loki_json`) lo permitiría — falta el parseo en Promtail.
- Imágenes de Loki, Promtail y Grafana sin versión fijada (`:latest`), ver [vps.md](vps.md).
- Retención fija de 7 días, sin diferenciación por nivel o servicio.
