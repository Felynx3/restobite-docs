# VPS

No hay repositorio propio para el VPS ni Terraform que lo aprovisione (ver [terraform-aws-gcp.md](terraform-aws-gcp.md)) — su configuración se reconstruye a partir de los scripts de `restobite-nginx`, `restobite-logs` y `restobite-backend`. Versión de Docker instalada en el host: `desconocido` (no versionada en ningún repo).

## Qué corre ahí

Todos los servicios se lanzan con `docker run` manuales por script (no hay `docker-compose` central para el VPS):

| Contenedor | Repo/origen del launcher | Imagen |
|---|---|---|
| `restobite-nginx` | `restobite-nginx/scripts/start.sh` | `nginx` (sin tag, efectivamente `latest`) |
| `restobite-backend` | `restobite-backend/scripts/*.sh` | build local desde `docker/node/DockerfileProd` |
| `restobite-public` | `restobite-public` (Dockerfiles/compose de prod) | build local desde `docker/DockerfileProd` |
| `jenkins` | no encontrado en ningún repo relevado | `desconocido` |
| `restobite-logs-loki` | `restobite-logs/loki/scripts/run-loki-container.sh` | `grafana/loki:latest` |
| `restobite-logs-promtail` | `restobite-logs/promtail/run-promtail-container.sh` | `grafana/promtail:latest` |
| `restobite-logs-grafana` | `restobite-logs/grafana/scripts/run-grafana-container.sh` | `grafana/grafana:latest` |

## Red

Todos los contenedores anteriores se unen a una red Docker externa `restobite-network`, creada manualmente (`docker network create restobite-network`, documentado en `restobite-logs/AGENTS.md`). El descubrimiento de servicios entre contenedores es por nombre de contenedor (p. ej. Nginx hace `proxy_pass` a `restobite-backend:4000`, `restobite-public:3000`, `jenkins:8080`, `restobite-logs-grafana:3000`).

## Volúmenes / paths de host compartidos

- `/root/restobite/data/nginx/logs` (host) → `/var/data/nginx/logs` en `restobite-nginx`, y el mismo path en `restobite-logs-promtail` — así llegan los logs de Nginx a Promtail.
- `/etc/letsencrypt` (host) → `/certs` en `restobite-nginx` (certificados TLS).
- Volumen Docker nombrado `restobite-logs-loki` → `/loki` en el contenedor de Loki (persistencia).
- `grafana/data/grafana` (path relativo al repo `restobite-logs` en el host) → `/var/lib/grafana` en Grafana.

## Puertos expuestos

- Nginx: `80` y `443` (únicos puertos públicos según `restobite-nginx/scripts/start.sh`).
- El resto de los contenedores (`backend:4000`, `restobite-public:3000`, `jenkins:8080`, `restobite-logs-grafana:3000`, Loki `3100`) solo son alcanzables dentro de `restobite-network`, vía Nginx.

## Cómo se actualiza cada servicio (estrategia de contenedor auxiliar)

- **Backend**: `restobite-backend/scripts/create-updated-aux-container.sh` + `update-docker-container.sh` — build de imagen nueva, se levanta un contenedor auxiliar en el puerto `4001` en paralelo al productivo (`4000`), se verifica que responde, se repunta la config de `restobite-nginx/configs/api.conf` al nuevo contenedor y se recarga Nginx (`docker exec restobite-nginx nginx -s reload`), y luego se elimina el contenedor reemplazado.
- **Frontend público**: existen Dockerfiles y `docker-compose.prod.yml` en el repo, pero no se encontró un script equivalente de contenedor auxiliar/swap. Mecanismo real de actualización en producción: `desconocido`.
- **Frontend administrativo**: no corre en el VPS — se despliega como sitio estático a S3 (ver [nivel-componente.md](../nivel-componente.md)).
- **Nginx, Loki, Promtail, Grafana**: los scripts de arranque (`start.sh` / `run-*-container.sh`) detienen y reemplazan el contenedor con el mismo nombre — no hay estrategia de cero downtime documentada para estos.
- **Jenkins**: forma de actualización `desconocido` (contenedor no definido en ningún repo relevado).

## Versionado de imágenes

`nginx`, `grafana/loki`, `grafana/promtail` y `grafana/grafana` se usan sin tag de versión fijado (`:latest` implícito). La imagen de `jenkins` es `desconocido`.
