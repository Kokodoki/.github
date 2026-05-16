## Why

El workflow `infra-terraform.yml` existente solo soporta autenticación en AWS. Los equipos que despliegan infraestructura en Azure carecen de un reusable workflow centralizado que ejecute las etapas de Terraform (validate → plan → apply) con autenticación nativa en Azure, lo que obliga a cada repositorio a mantener su propia implementación ad-hoc.

## What Changes

- Se agrega el reusable workflow `.github/workflows/infra-terraform-azure.yml` con jobs `validate`, `plan` y `apply` equivalentes al workflow de AWS, pero autenticando contra Azure mediante OIDC (Workload Identity Federation) o credenciales estáticas (client secret).
- Se agrega documentación en `docs/infra-terraform-azure.md` con propósito, ejemplo de uso, tabla de inputs/outputs/secrets y descripción de jobs.
- Se agrega workflow template compuesto por `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

## Capabilities

### New Capabilities

- `infra-terraform-azure`: Reusable workflow que ejecuta Terraform sobre Azure con autenticación OIDC o client secret, jobs validate/plan/apply con gate de aprobación vía GitHub Environment, comentario del plan en Pull Requests y subida del artefacto `tfplan`.

### Modified Capabilities

<!-- Sin cambios en capabilities existentes -->

## Impact

- **Nuevo archivo**: `.github/workflows/infra-terraform-azure.yml`
- **Nueva documentación**: `docs/infra-terraform-azure.md`
- **Nuevos templates**: `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`
- **Dependencias externas**:
  - `hashicorp/setup-terraform@v4` (ya usada en el workflow de AWS)
  - `azure/login@v2` (action oficial de Microsoft para autenticar en Azure)
  - `actions/checkout`, `actions/upload-artifact`, `actions/download-artifact`, `actions/github-script` (ya usadas en el workflow de AWS)
- **Secrets**: puede requerir secrets a nivel de organización (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`) o a nivel de repositorio según el modelo de governance de la organización; el workflow los acepta en ambos niveles.
- **Sin cambios en workflows existentes**: el workflow de AWS (`infra-terraform.yml`) no se modifica.
