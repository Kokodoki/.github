## Why

Los repositorios de la organización que construyen imágenes Docker duplican actualmente la lógica de build en cada uno de sus workflows. Centralizar esta lógica en un reusable workflow elimina la duplicación, garantiza consistencia en el proceso de build y facilita el mantenimiento futuro.

## What Changes

- Se crea el reusable workflow `.github/workflows/build-docker.yml` con `on: workflow_call`, exponiendo inputs para imagen, tag, contexto, Dockerfile y plataforma objetivo.
- Se crea la documentación `/docs/build-docker.md` con descripción del workflow, inputs/outputs/secrets y ejemplo de consumo.
- Se crea el workflow template `/workflow-templates/build-docker.yml` y su archivo de metadatos `/workflow-templates/build-docker.properties.json` para facilitar la adopción desde otros repositorios.

## Capabilities

### New Capabilities

- `build-docker`: Reusable workflow que encapsula el proceso completo de construcción de imágenes Docker, incluyendo checkout, configuración de QEMU/Buildx, autenticación en el registry (configurable) y publicación de la imagen resultante.

### Modified Capabilities

## Impact

- **Repositorios consumidores**: Cualquier repositorio de la organización que construya imágenes Docker; pueden migrar su lógica de build apuntando a este workflow.
- **GitHub Actions**: Se utiliza la sintaxis `on: workflow_call`. Requiere actions de terceros: `actions/checkout`, `docker/setup-qemu-action`, `docker/setup-buildx-action`, `docker/login-action`, `docker/build-push-action`.
- **Secrets**: El workflow requerirá secrets para autenticación en el registry (e.g., `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`) pasados desde el repositorio consumidor o a nivel de organización.
- **Sin breaking changes** para workflows existentes; la adopción es opt-in.
