# restobite-docs

Fuente de verdad de arquitectura de Restobite, orientada a agentes de IA. Documenta el **estado actual verificado** del sistema (no un estado deseado). Cualquier cambio arquitectónico en Restobite debe reflejarse actualizando este repositorio.

Cuando un dato no pudo verificarse en código, configuración o infraestructura real, se marca explícitamente como `desconocido` en el lugar donde correspondería — no se completa por inferencia.

## Cómo navegar este repositorio

1. **[mapa-componentes.md](mapa-componentes.md)** — vista general de todos los componentes del sistema y cómo se conectan entre sí. Punto de entrada obligatorio antes de leer cualquier otro archivo.
2. **[nivel-componente.md](nivel-componente.md)** — resumen (propósito, estado, relaciones, autenticación, despliegue) de backend, frontend público y frontend administrativo. Sin listado de endpoints ni contratos de API.
3. **`infraestructura/`** — detalle de infraestructura:
   - [terraform-aws-gcp.md](infraestructura/terraform-aws-gcp.md)
   - [vps.md](infraestructura/vps.md)
   - [nginx.md](infraestructura/nginx.md)
   - [redes-dns.md](infraestructura/redes-dns.md)
   - [persistencia.md](infraestructura/persistencia.md)
   - [integraciones-externas.md](infraestructura/integraciones-externas.md)
   - [observabilidad.md](infraestructura/observabilidad.md)
   - [cicd.md](infraestructura/cicd.md)
4. **[flujos-end-to-end.md](flujos-end-to-end.md)** — los flujos completos (navegación pública, login/edición administrativa, carga de assets, despliegue, logs) como secuencia de componentes.

## Repositorios cubiertos

`restobite-backend`, `restobite-admin`, `restobite-public`, `restobite-cloud`, `restobite-nginx`, `restobite-logs`. Cada uno mantiene su propio `AGENTS.md` con convenciones de desarrollo (estructura, comandos, estilo); ese contenido no se repite acá.

## Fuera de alcance

Rutas/endpoints uno por uno, contratos de API detallados, cobertura de pruebas archivo por archivo, y cualquier recomendación o arquitectura objetivo no verificada en el estado actual.
