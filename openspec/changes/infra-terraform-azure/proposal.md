## Why

Los repositorios de la organización que gestionan infraestructura en **Azure** carecen de un mecanismo centralizado y seguro para ejecutar flujos de Terraform. Se necesita un reusable workflow que estandarice la autenticación mediante OIDC, la validación, la planificación con comentarios en Pull Requests y la aplicación controlada con aprobación manual, igual que el workflow existente para AWS (`infra-terraform`).

## What Changes

- Nuevo reusable workflow `infra-terraform-azure.yml` en `.github/workflows/` con los jobs `validate`, `plan` y `apply`.
- Autenticación en Azure mediante OIDC usando `azure/login` (sin secretos de credenciales estáticas en el caso básico; solo `client-id`, `tenant-id` y `subscription-id`).
- Job `validate` que ejecuta `terraform fmt -check -recursive`, `terraform init -backend=false` y `terraform validate`.
- Job `plan` que ejecuta `terraform init` con backend real, `terraform plan`, sube el artifact `tfplan` y comenta el output en Pull Requests.
- Job `apply` condicionado al input booleano `enable-apply`, protegido por un GitHub Environment para aprobación manual.
- Documentación en `docs/infra-terraform-azure.md`.
- Workflow template en `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

## Capabilities

### New Capabilities

- `infra-terraform-azure`: Reusable workflow para ejecutar el ciclo de vida completo de Terraform sobre Azure con OIDC, gate de aprobación y comentarios de plan en PRs.

### Modified Capabilities

<!-- ninguna -->

## Impact

- **Nuevos archivos:** `.github/workflows/infra-terraform-azure.yml`, `docs/infra-terraform-azure.md`, `workflow-templates/infra-terraform-azure.yml`, `workflow-templates/infra-terraform-azure.properties.json`.
- **Dependencias externas:** `actions/checkout`, `hashicorp/setup-terraform`, `azure/login`, `actions/upload-artifact`, `actions/download-artifact`, `actions/github-script`.
- **Secrets requeridos a nivel de organización o repositorio:** `azure-client-secret` (opcional, solo si no se usa OIDC puro).
- **Inputs:** `terraform-version`, `working-directory`, `azure-client-id`, `azure-tenant-id`, `azure-subscription-id`, `environment-name`, `backend-config`, `enable-apply`.
- **Outputs:** `plan-exit-code`.
