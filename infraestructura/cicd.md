# CI/CD

Jenkins es central al despliegue de Restobite, pero su configuración vive mayormente **fuera de Git** — no existe un repositorio de Jenkins, `Jenkinsfile` ni definición del contenedor de Jenkins en ninguno de los 6 repos relevados (`restobite-backend`, `restobite-admin`, `restobite-public`, `restobite-cloud`, `restobite-nginx`, `restobite-logs`).

## Lo que sí está verificado

- **Identidad/permisos**: `restobite-cloud` (módulo `ci/`) declara un usuario IAM `aws_iam_user.jenkins` con policies inline de S3 (subir/borrar sobre el bucket admin-app, listar todos los buckets) y un `aws_iam_access_key` asociado. Esto es lo único que Terraform gestiona respecto a Jenkins.
- **Red/ruteo**: `restobite-nginx` (`configs/jenkins.conf`) enruta `ci.restobite.com` → `http://jenkins:8080`, y Terraform (`restobite-cloud`, módulo `ci/`) apunta ese dominio, vía Route 53, a la IP del VPS. Esto implica que un contenedor llamado `jenkins` corre en `restobite-network`, en el mismo VPS que Nginx — pero ese contenedor no está definido (ni por Dockerfile, ni por `docker run`, ni por compose) en ningún repo relevado.
- **Scripts de build/deploy por repo** (probablemente lo que Jenkins invoca, sin evidencia directa de que sea Jenkins quien los ejecuta):
  - `restobite-backend/scripts/create-updated-aux-container.sh` + `update-docker-container.sh`: build de imagen, contenedor auxiliar, swap, reload de Nginx.
  - `restobite-admin/scripts/build.sh` (build en contenedor `node:22`) + `scripts/deploy.sh` (`aws s3 sync` directo al bucket S3 `restobite-admin-app-prod`).
  - `restobite-public/scripts/build.sh`: build de imagen Docker con `docker/DockerfileProd`; no se encontró script de despliegue/swap posterior a ese build.

## Incógnitas verificables (preguntas concretas, no se asume respuesta)

1. ¿Qué imagen, puertos y volúmenes usa el contenedor `jenkins` en el VPS? ¿Dónde está definido (compose/script no versionado, o creado a mano)?
2. ¿Qué jobs de Jenkins existen hoy, uno por repo o compartidos? ¿Qué rama/evento dispara cada uno (push a `main`, tag, manual)?
3. ¿Cada job de Jenkins solo invoca los scripts ya listados arriba (`create-updated-aux-container.sh`, `deploy.sh`, `build.sh`, etc.), o hay pasos adicionales definidos únicamente en la configuración de Jenkins (no versionados)?
4. ¿Existe algún `Jenkinsfile` como código, guardado localmente en el VPS o solo configurado vía la UI de Jenkins (pipeline no declarativo en Git)?
5. ¿Cómo se almacenan y usan dentro de Jenkins las credenciales del usuario IAM `jenkins` (access key/secret) — como "credential" de Jenkins, variable de entorno del contenedor, u otro mecanismo?
6. ¿Cómo se actualiza en producción el contenedor `restobite-public` una vez construida la imagen? (`restobite-backend` tiene un mecanismo de swap explícito; no se encontró equivalente para `restobite-public`.)
7. ¿Hay trigger automático (webhook de GitHub/GitLab) hacia Jenkins en cada push, o el disparo es manual?
8. ¿Quién administra/tiene acceso a `ci.restobite.com` (usuarios de Jenkins, control de acceso)?
