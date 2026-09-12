# Nginx

Repositorio: `restobite-nginx`. `nginx.conf` (raíz) define el formato de log `loki_json` y hace `include` de cada archivo en `configs/`.

## Sitios configurados

| Archivo | `server_name` | `proxy_pass` | Notas |
|---|---|---|---|
| `configs/api.conf` | `origin-api.restobite.com` | `http://restobite-backend:4000` | Los bloques 80 y 443 validan un header `X-Origin-Secret` que debe coincidir con el que envía CloudFront (módulo `backend` de Terraform); valor no reproducido. |
| `configs/jenkins.conf` | `ci.restobite.com` | `http://jenkins:8080` | Redirect HTTP→HTTPS; sin validación de `X-Origin-Secret`. |
| `configs/logs.conf` | `logs.restobite.com` | `http://restobite-logs-grafana:3000` | Sin validación de `X-Origin-Secret`. |
| `configs/public.conf` | `origin-www.restobite.com` | `http://restobite-public:3000` | Valida `X-Origin-Secret` (valor distinto al de `api.conf`); access/error log en formato `loki_json` a `/var/data/nginx/logs/restobite_public.log` y `..._error.log`. |

## TLS / certificados

Todos los bloques HTTPS referencian rutas estilo Let's Encrypt: `/certs/live/<dominio>/fullchain.pem` y `/certs/live/<dominio>/privkey.pem`. `/certs` está montado desde `/etc/letsencrypt` del host (ver [vps.md](vps.md)). No se encontró script de renovación (certbot) ni cron en este repositorio — la emisión/renovación de certificados es externa a este repo. `desconocido` dónde y cómo se ejecuta esa renovación.

## Reglas de proxy

Cada sitio hace `proxy_pass` a un contenedor por nombre dentro de `restobite-network` (ver [vps.md](vps.md)). Los orígenes que reciben tráfico desde CloudFront (`api.conf`, `public.conf`) exigen el header secreto; los que no pasan por CloudFront (`jenkins.conf`, `logs.conf`) no lo exigen, ya que reciben tráfico directo por IP del VPS (ver [redes-dns.md](redes-dns.md)).

## Arranque

`scripts/start.sh` levanta el contenedor con `docker run -p 80:80 -p 443:443`, montando `nginx.conf`, `configs/`, los logs (`/root/restobite/data/nginx/logs`) y los certificados (`/etc/letsencrypt`), uniéndose a `restobite-network`, con `--restart unless-stopped`. Imagen: `nginx` sin tag fijado.
