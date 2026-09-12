# Redes y DNS

Gestionado por Route 53 (Terraform, repo `restobite-cloud`). Valores concretos (IP del VPS, Zone ID) no se reproducen acá; se documenta la estructura de apuntamiento.

## Dominios y a qué apuntan

| Dominio | Apunta a | Vía |
|---|---|---|
| `admin.restobite.com` | S3 (`restobite-admin-app-prod`) | CloudFront, alias Route 53 |
| `api.restobite.com` | VPS (contenedor backend) | CloudFront → `origin-api.restobite.com` (A record directo a la IP del VPS) |
| `www.restobite.com` | VPS (contenedor frontend público) | CloudFront → `origin-www.restobite.com` (A record directo a la IP del VPS) |
| `restobite.com` (raíz) | S3 (bucket de redirect) | CloudFront → 301 a `www.restobite.com` |
| `assets.restobite.com` | S3 (`public_assets`) | CloudFront, alias Route 53 |
| `ci.restobite.com` | VPS (contenedor Jenkins) | A record directo a la IP del VPS, sin CloudFront |
| `logs.restobite.com` | VPS (contenedor Grafana) | A record directo a la IP del VPS, sin CloudFront |
| `mail.restobite.com` / registros MX-SPF-DKIM-DMARC | SES | Registros gestionados por el módulo `email` de Terraform |

## Certificados TLS

- Certificados para los dominios servidos por CloudFront (`admin`, `api` público, `www` público, `assets`, redirect de raíz): `aws_acm_certificate` + validación DNS vía Route 53, por módulo de Terraform.
- Certificados para los orígenes servidos directo por el VPS (`ci.restobite.com`, `logs.restobite.com`, y los hosts `origin-*` que Nginx termina): Let's Encrypt, gestionados fuera de Terraform (ver [nginx.md](nginx.md)).

## Notas

- Los orígenes `origin-api.restobite.com` y `origin-www.restobite.com` existen específicamente para que CloudFront tenga un origen HTTPS estable apuntando al VPS, distinto del dominio público final.
- No se encontró ningún registro DNS wildcard (`*.restobite.com`) en los módulos relevados — no hay evidencia de subdominios por tenant a nivel DNS.
