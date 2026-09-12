# Terraform / AWS / GCP

Repositorio: `restobite-cloud`. No se leyó ni se reproduce contenido de `terraform.tfstate`, `terraform.tfstate.backup`, `secrets.tfvars` ni de directorios `credentials/` (regla de seguridad del propio `AGENTS.md` de ese repo). Lo siguiente es inventario de recursos **declarados** en archivos `.tf` regulares, no confirmación de que coincidan 1:1 con el estado real en las cuentas.

## Qué gestiona Terraform

Un root module en la raíz de `restobite-cloud` instancia 8 módulos:

| Módulo | Recursos declarados (tipos) |
|---|---|
| `admin-app/` | `aws_s3_bucket`, `aws_s3_bucket_website_configuration`, `aws_s3_bucket_policy` + `data.aws_iam_policy_document`, `aws_cloudfront_distribution`, `aws_cloudfront_origin_access_control`, `aws_acm_certificate` (+ validación), `aws_route53_record` (A/AAAA + validación) |
| `backend/` | `aws_cloudfront_distribution` (con header custom `X-Origin-Secret`), `aws_cloudfront_cache_policy`, `aws_cloudfront_origin_request_policy`, `aws_acm_certificate`, `aws_route53_record` (origin A hacia la IP del VPS, y alias público hacia CloudFront) |
| `ci/` | `aws_route53_record` (`ci.restobite.com` → IP del VPS), `aws_iam_user.jenkins`, `aws_iam_user_policy` (S3 put/delete/list sobre el bucket admin-app y list-all-buckets), `aws_iam_access_key.jenkins_access_key` |
| `email/` | `aws_ses_domain_identity`, `aws_ses_domain_mail_from`, `aws_route53_record` (MX/SPF/DKIM/DMARC), `aws_ses_domain_dkim`, `aws_ses_configuration_set`, `aws_ses_receipt_rule_set`, `aws_ses_receipt_rule` (por destinatario), `aws_ses_active_receipt_rule_set`, `aws_s3_bucket` + `aws_s3_bucket_policy` (buzón de SES) |
| `logs/` | `aws_route53_record` (`logs.restobite.com` → IP del VPS) |
| `public-page/` | `aws_cloudfront_distribution` (con `X-Origin-Secret`), `aws_cloudfront_response_headers_policy`, múltiples `aws_cloudfront_cache_policy`/`origin_request_policy`, `aws_acm_certificate` + validación, `aws_route53_record` (origin A hacia el VPS y alias público hacia CloudFront); además redirect de raíz: `aws_s3_bucket` (bucket `restobite.com`), `aws_s3_bucket_website_configuration` (redirect a `www.restobite.com`), `aws_s3_bucket_public_access_block`, `aws_s3_bucket_policy`, `aws_cloudfront_distribution` (redirect), `aws_acm_certificate` + validación, `aws_route53_record` (raíz) |
| `public_assets/` | `aws_s3_bucket`, `aws_s3_bucket_cors_configuration`, `aws_s3_bucket_policy`, `aws_cloudfront_distribution` (sirve `assets.restobite.com`), `aws_cloudfront_response_headers_policy`, `aws_cloudfront_cache_policy`, `aws_cloudfront_origin_request_policy`, `aws_cloudfront_origin_access_control`, `aws_acm_certificate` + validación, `aws_route53_record` (A/AAAA) |
| `gcloud/` | `google_recaptcha_enterprise_key` (prod y dev), submódulo `database/`: `google_sql_database_instance` (Postgres 15), `google_sql_user` (admin, backend); submódulo `credentials/` (no inspeccionado, contiene secretos) |

Root: `provider "aws"` en `us-east-1`; locals con la IP del VPS y el Zone ID de Route 53 (valores no reproducidos).

## Qué NO gestiona Terraform

No existe ningún recurso `aws_instance` en el repositorio: la IP del VPS es una entrada externa (`var.vps_ip`), no algo que Terraform aprovisione. Terraform solo apunta registros DNS hacia esa IP. El VPS en sí, Docker, Nginx, Jenkins (el contenedor) y el stack de logs (Loki/Promtail/Grafana) están definidos y gestionados en los repos `restobite-nginx` y `restobite-logs` mediante scripts, no en `restobite-cloud`. No existe `README.md` en `restobite-cloud` que documente explícitamente qué se creó a mano versus por Terraform; esa distinción se infiere de la ausencia de `aws_instance` y de recursos para nginx/Jenkins/logs.

## Estado de los `.tfstate`

Existen archivos `terraform.tfstate`/`.backup` en el repositorio (no leídos, por regla de seguridad). No se verificó backend remoto de state (S3+DynamoDB u otro) — `desconocido`.

## Cuentas y proyectos involucrados

- AWS: una cuenta, región `us-east-1` (recursos de `admin-app`, `backend`, `ci`, `email`, `public-page`, `public_assets`).
- GCP: proyecto `restobite-415221`, región `southamerica-west1` (Cloud SQL, reCAPTCHA Enterprise).

## `ci/` — relación con Jenkins

El módulo `ci/` no declara un servidor Jenkins: crea un **usuario IAM** (`aws_iam_user.jenkins`) con policies inline de S3 y un access key asociado, más el registro DNS `ci.restobite.com` apuntando al VPS. Es decir, Terraform sólo le da a Jenkins credenciales AWS y un nombre DNS; el contenedor Jenkins en sí corre en el VPS fuera de este repositorio (ver [cicd.md](cicd.md)).

## `gcloud/` — Cloud SQL

`google_sql_database_instance.restobite` (Postgres 15, base `restobite-prod`, `southamerica-west1`, `deletion_protection = true`, backups deshabilitados en el código relevado) con dos usuarios (`admin`, `backend`). No hay comentario/README que la describa como alternativa a una base AWS: es la única base de datos relacional declarada en todo el repositorio — no existe ningún recurso RDS en el lado AWS. El backend en el VPS se conecta a esta instancia por red autorizada (allowlist de IPs, incluyendo la IP del VPS).

## Dominios/URLs que revelan topología de despliegue (no sensibles)

- `admin.restobite.com` → CloudFront → S3 (`restobite-admin-app-prod`)
- `api.restobite.com` → CloudFront → `origin-api.restobite.com` → VPS
- `www.restobite.com` → CloudFront → `origin-www.restobite.com` → VPS
- `restobite.com` (raíz) → S3 (redirect) → CloudFront → 301 a `www.restobite.com`
- `assets.restobite.com` → CloudFront → S3 (`public_assets`)
- `ci.restobite.com` → VPS directo (sin CloudFront)
- `logs.restobite.com` → VPS directo (sin CloudFront)
- Dominio de correo SES: `restobite.com` (MX/SPF/DKIM/DMARC en `mail.restobite.com`)
