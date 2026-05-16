## Why

El repositorio ya cuenta con `infra-terraform.yml` para infraestructura en AWS, pero no existe un equivalente para Azure. Los equipos que despliegan infraestructura en Azure con Terraform deben mantener sus propios workflows sin estándares organizacionales, lo que genera inconsistencias en autenticación, validación y aprobación de cambios.

## What Changes

- Se agrega el reusable workflow `infra-terraform-azure.yml` con los jobs `validate`, `plan` y `apply` adaptados para Azure.
- Se soportan dos métodos de autenticación en Azure: OIDC (Workload Identity Federation) y Service Principal con credenciales estáticas.
- El backend de Terraform se configura sobre Azure Blob Storage (en lugar de S3/DynamoDB).
- Se publica el output de `terraform plan` como comentario en Pull Requests.
- El job `apply` requiere aprobación manual mediante GitHub Environment.
- Se agrega documentación en `docs/infra-terraform-azure.md`.
- Se agrega workflow template en `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

## Capabilities

### New Capabilities

- `infra-terraform-azure`: Reusable workflow de GitHub Actions que ejecuta el ciclo completo de Terraform (validate → plan → apply) autenticando en Azure mediante OIDC o Service Principal, usando Azure Blob Storage como backend de estado, comentando el plan en PRs y requiriendo aprobación manual vía GitHub Environment antes del apply.

### Modified Capabilities

## Impact

- **Nuevo archivo**: `.github/workflows/infra-terraform-azure.yml`
- **Nueva documentación**: `docs/infra-terraform-azure.md`
- **Nuevo template**: `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`
- **Dependencias externas**: `azure/login@v2`, `hashicorp/setup-terraform@v4`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`, `actions/github-script@v9`
- **Secrets requeridos**: `azure-client-id`, `azure-client-secret`, `azure-tenant-id`, `azure-subscription-id` (opcionales según método de autenticación)
- **Sin cambios en workflows existentes**; este workflow es completamente independiente de `infra-terraform.yml`
