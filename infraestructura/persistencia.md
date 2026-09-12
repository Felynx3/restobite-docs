# Persistencia

Repositorio: `restobite-backend` (schema y migraciones), instancia gestionada en GCP (repo `restobite-cloud`, ver [terraform-aws-gcp.md](terraform-aws-gcp.md)).

## Motor y ubicación

PostgreSQL 15, instancia `google_sql_database_instance.restobite` en GCP Cloud SQL (proyecto `restobite-415221`, región `southamerica-west1`, base `restobite-prod`). No existe ningún recurso RDS en AWS: esta es la única base de datos relacional del sistema, no una alternativa/paralela. Conexión desde el backend vía una única variable `DATABASE_URL` (`prisma/schema.prisma`, `datasource db`); en desarrollo local apunta a un contenedor Postgres del propio `docker-compose.yml` del backend, no a Cloud SQL.

## ORM y migraciones

Prisma. Comandos: `yarn prisma:generate`, `yarn prisma:migrate-dev` (desarrollo), `yarn prisma:migrate` (deploy de migraciones). Los archivos de schema, seeds y migraciones viven bajo `prisma/` en `restobite-backend`. Contenido específico de los seeds: `desconocido` (no relevado en detalle).

## Estrategia de multitenancy

Aislamiento **por fila**, no por schema ni por base de datos separada: casi todos los modelos de `prisma/schema.prisma` tienen una columna `tenantId` con FK a un único modelo `Tenant` compartido.

El aislamiento se refuerza a nivel de Prisma Client, no con un interceptor de NestJS:

- Una extensión de Prisma Client (`tenant-prisma.service.factory.ts`) intercepta toda operación (`$allModels.$allOperations`).
- Para cada operación de lectura/escritura (`count`, `delete(Many)`, `find*`, `update(Many)`, `upsert`), valida que el `where` de la query incluya un `tenantId` (o `tenant.id`, o una clave anidada que contenga `tenantId`) igual al tenant activo en el contexto de la request (CLS). Si no coincide, lanza una excepción y bloquea la query.
- Existe una lista explícita de modelos exceptuados de este chequeo (tablas de relación y modelos de autenticación/roles compartidos: `CustomizationGroupToMenuItem`, `CustomizationGroupToMenuCategory`, `StaticRole`, `Credential`, `PasswordAuthentication`).

## Cómo se determina el tenant activo

- En requests autenticadas con JWT (frontend administrativo): el tenant viene del payload del token y se setea en el contexto (CLS) al validar el JWT.
- En rutas públicas de menú: un middleware global resuelve el tenant por código a partir de la URL y lo setea en el contexto. Se observó una inconsistencia entre el string que este middleware busca en la URL (`menu/public/`) y el path real montado del controller de menú público (`public/tenants/:tenantCode/menus/`) — se reporta como observación de código, sin interpretar más allá de lo que el código muestra.

## Invalidación

Existe un mecanismo de invalidación de tenant (`@InvalidatesTenant()`, interceptor + servicio dedicado) que llama a una URL externa configurable (`TENANT_INVALIDATION_URL_TEMPLATE`) tras ciertas mutaciones, para invalidar cachés de tenant fuera del backend. Qué consume esa URL: `desconocido`.
