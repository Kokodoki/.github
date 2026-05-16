## Why

Los repositorios Java de la organización no tienen un reusable workflow estándar para compilar con Maven y publicar artefactos en JFrog, lo que genera pipelines inconsistentes, duplicación y errores de configuración. Estandarizar este flujo ahora reduce tiempos de adopción y mejora trazabilidad en las entregas.

## What Changes

- Crear un reusable workflow `.github/workflows/build-maven-jfrog.yml` con `on: workflow_call` para compilar proyectos Maven y publicar artefactos en JFrog Artifactory.
- Definir inputs en kebab-case para configuración de Java, Maven, ruta de proyecto, repositorio objetivo en JFrog y parámetros de publicación.
- Definir secrets en kebab-case para autenticación contra JFrog (usuario/token o API key) y documentar su alcance (repositorio u organización).
- Exponer outputs en kebab-case con metadatos de build/publicación (por ejemplo, coordenadas publicadas y URL de referencia).
- Incluir documentación en `docs/build-maven-jfrog.md` con propósito, ejemplo de consumo, inputs/outputs/secrets y detalle de jobs/steps.
- Incluir workflow template en `workflow-templates/build-maven-jfrog.yml` y `workflow-templates/build-maven-jfrog.properties.json` para facilitar adopción desde la UI de GitHub.
- Documentar dependencias externas requeridas (actions de GitHub, JDK, Maven y endpoint de JFrog Artifactory).
- Especificar casos de uso beneficiados: repositorios de librerías Java y servicios Java que publican artefactos en repositorios Maven internos de JFrog.

## Capabilities

### New Capabilities
- `build-maven-jfrog`: reusable workflow para compilar aplicaciones Maven y publicar artefactos en JFrog con una interfaz estándar de inputs/outputs/secrets.

### Modified Capabilities
- Ninguna.

## Impact

- Archivos afectados: `.github/workflows/`, `docs/`, `workflow-templates/`, y nuevos specs del cambio.
- Sistemas impactados: GitHub Actions (workflow_call) y JFrog Artifactory.
- Dependencias externas: `actions/checkout`, `actions/setup-java`, herramientas Maven/JDK y credenciales JFrog.
- Seguridad/operación: requiere manejo de secrets para autenticación en JFrog y políticas de permisos en repositorios consumidores.
