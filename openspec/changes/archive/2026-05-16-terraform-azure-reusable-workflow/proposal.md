## Why

El repositorio ya cuenta con un reusable workflow de Terraform para AWS (`infra-terraform.yml`), pero no existe un equivalente para Azure. Los equipos que usan Terraform con Azure Resource Manager no tienen un workflow estándar centralizado, lo que genera duplicación y falta de consistencia entre repositorios.

## What Changes

- **Nuevo reusable workflow** `infra-terraform-azure.yml` en `.github/workflows/` que ejecuta los jobs `validate`, `plan` y `apply` para infraestructura Terraform sobre Azure.
- **Autenticación Azure** mediante OIDC (Federated Identity Credentials) o Service Principal con client secret, usando `azure/login@v2`.
- **Backend Azure Storage** con soporte para configurar `resource-group-name`, `storage-account-name`, `container-name` y `key` del estado remoto.
- **Comentario de plan en Pull Requests** con el output de `terraform plan` usando `permissions: pull-requests: write`.
- **Gate de aprobación manual** en el job `apply` mediante GitHub Environment.
- **Documentación** en `docs/infra-terraform-azure.md` con propósito, ejemplo de uso, tabla de inputs/outputs/secrets y descripción de jobs.
- **Workflow template** en `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json` para facilitar la adopción desde la UI de GitHub.

## Capabilities

### New Capabilities

- `infra-terraform-azure`: Reusable workflow de GitHub Actions que ejecuta el ciclo completo de Terraform (validate → plan → apply) autenticando contra Azure mediante OIDC o Service Principal, con backend en Azure Storage, gate de aprobación manual y comentario de plan en PRs.

### Modified Capabilities

## Impact

- **Nuevos archivos**: `.github/workflows/infra-terraform-azure.yml`, `docs/infra-terraform-azure.md`, `workflow-templates/infra-terraform-azure.yml`, `workflow-templates/infra-terraform-azure.properties.json`
- **Repositorios beneficiados**: todos los que usen Terraform con Azure Resource Manager
- **Dependencias externas**: `azure/login@v2`, `hashicorp/setup-terraform@v4`, `actions/checkout@v6`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`, `actions/github-script@v9`
- **Secrets requeridos**: `azure-client-id` (OIDC) o `azure-client-id` + `azure-client-secret` (Service Principal); `azure-tenant-id` y `azure-subscription-id` siempre requeridos
- **Secrets de nivel de organización o repositorio**: `azure-tenant-id` y `azure-subscription-id` pueden configurarse a nivel organización; `azure-client-id` y `azure-client-secret` a nivel repositorio o entorno
