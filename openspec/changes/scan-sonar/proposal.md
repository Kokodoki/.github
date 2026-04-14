## Why

Los repositorios de la organización no cuentan con un mecanismo centralizado para analizar la calidad del código con SonarCloud ni para verificar el Quality Gate de forma automática en los pipelines de CI. Estandarizar este proceso como un reusable workflow permite adoptarlo en cualquier repositorio con mínima configuración.

## What Changes

- Se crea el reusable workflow `scan-sonar.yml` en `.github/workflows/` con `on: workflow_call`.
- El workflow ejecuta el análisis de código con SonarCloud y valida que el Quality Gate pase antes de permitir continuar el pipeline.
- Se crea la documentación del workflow en `/docs/scan-sonar.md`.
- Se crea el workflow template en `/workflow-templates/scan-sonar.yml` y su archivo de metadatos `/workflow-templates/scan-sonar.properties.json`.

## Capabilities

### New Capabilities

- `scan-sonar`: Reusable workflow que ejecuta el scanner de SonarCloud sobre el código fuente del repositorio consumidor y valida el Quality Gate. Expone inputs para configurar el proyecto de Sonar, la rama principal y opciones del scanner; requiere un secret con el token de SonarCloud.

### Modified Capabilities

<!-- Sin cambios en capabilities existentes -->

## Impact

- **Workflows afectados**: ninguno existente; se agrega `scan-sonar.yml` nuevo.
- **Dependencias externas**:
  - Action `SonarSource/sonarcloud-github-action` (análisis).
  - SonarCloud (servicio SaaS externo).
- **Secrets requeridos**: `SONAR_TOKEN` — debe existir a nivel de organización o repositorio en cada consumidor.
- **Inputs**:
  - `sonar-project-key` (string, requerido): clave del proyecto en SonarCloud.
  - `sonar-organization` (string, requerido): organización en SonarCloud.
  - `main-branch` (string, opcional, default `main`): rama principal del repositorio.
  - `extra-args` (string, opcional): argumentos adicionales para el scanner.
- **Outputs**: ninguno (el workflow falla si el Quality Gate no pasa).
- **Repositorios beneficiados**: cualquier repositorio de la organización que desee integrar análisis de calidad con SonarCloud en su CI.
