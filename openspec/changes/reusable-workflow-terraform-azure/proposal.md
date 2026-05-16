## Why

La organización ya cuenta con un reusable workflow de Terraform para AWS (`infra-terraform`), pero no dispone de uno equivalente para Azure. Los equipos que despliegan infraestructura en Azure requieren un workflow reutilizable que soporte autenticación segura mediante OIDC (sin credenciales estáticas), incluya las mismas garantías de calidad (validate → plan → apply) y publique el resultado del plan en Pull Requests para revisión antes de aplicar cambios.

## What Changes

- **Nuevo workflow** `.github/workflows/infra-terraform-azure.yml`: reusable workflow con jobs `validate`, `plan` y `apply` para Terraform sobre Azure usando OIDC.
- **Autenticación OIDC exclusiva** con `azure/login@v2` usando los inputs `azure-client-id`, `azure-tenant-id` y `azure-subscription-id`; no se soportan credenciales estáticas.
- **Job validate**: ejecuta `terraform fmt -check -recursive`, `terraform init -backend=false` y `terraform validate` antes de plan y apply.
- **Job plan**: ejecuta `terraform init` con backend real, `terraform plan -out=tfplan`, sube el artifact `tfplan` y publica/actualiza un comentario en el PR con el output del plan (sólo en eventos `pull_request`).
- **Job apply**: descarga el artifact `tfplan`, requiere aprobación manual a través del GitHub Environment indicado y ejecuta `terraform apply tfplan`; está condicionado por el input booleano `enable-apply`.
- **Nueva documentación** en `docs/infra-terraform-azure.md`.
- **Nuevo workflow template** en `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

## Capabilities

### New Capabilities

- `infra-terraform-azure`: Reusable workflow de Terraform para Azure con autenticación OIDC, jobs validate/plan/apply, comentario de plan en PRs y gate de aprobación en apply mediante GitHub Environments.

### Modified Capabilities

_Ninguna. El cambio no modifica los requisitos del workflow existente `infra-terraform` (AWS)._

## Impact

- **Nuevo archivo**: `.github/workflows/infra-terraform-azure.yml`
- **Nueva documentación**: `docs/infra-terraform-azure.md`
- **Nuevos templates**: `workflow-templates/infra-terraform-azure.yml`, `workflow-templates/infra-terraform-azure.properties.json`
- **Nuevo spec**: `openspec/specs/infra-terraform-azure/spec.md`
- **Dependencias externas**:
  - `azure/login@v2` (action oficial de Microsoft para autenticación en Azure)
  - `hashicorp/setup-terraform@v4` (ya usado en el workflow AWS)
  - `actions/upload-artifact@v4`, `actions/download-artifact@v4`, `actions/github-script@v7` (ya usados en el workflow AWS)
- **Secrets/permisos requeridos**:
  - Permiso `id-token: write` para OIDC
  - Permiso `pull-requests: write` para comentarios en PR
  - Secrets a nivel de organización o repositorio: ninguno (OIDC usa inputs, no secrets estáticos)
  - Los inputs `azure-client-id`, `azure-tenant-id` y `azure-subscription-id` deben ser provistos por el repositorio consumidor (típicamente desde variables de entorno o vars del repositorio)
