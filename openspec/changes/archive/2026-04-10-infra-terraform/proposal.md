## Why

Los equipos de la organización necesitan aprovisionar infraestructura en AWS con Terraform de forma estandarizada, segura y repetible desde sus pipelines de CI/CD. Existe un patrón común de validación → plan → apply que hoy cada repositorio implementa de forma distinta, generando inconsistencias y duplicación.

## What Changes

- Se introduce el reusable workflow `infra-terraform.yml` en `.github/workflows/` con tres jobs orquestados: `validate`, `plan` y `apply`.
- El job `validate` ejecuta `terraform fmt`, `terraform init -backend=false` y `terraform validate`.
- El job `plan` ejecuta `terraform init` con backend real y `terraform plan`, exportando el `tfplan` como artifact de GitHub Actions.
- El job `plan` comenta el output del plan en el Pull Request cuando el workflow es disparado por un evento de PR, facilitando la revisión del cambio de infraestructura.
- El job `apply` descarga el `tfplan` generado en `plan`, requiere un GitHub Environment para aprobación manual y ejecuta `terraform apply`.
- Se crea la documentación del workflow en `docs/infra-terraform.md`.
- Se crea el workflow template en `workflow-templates/infra-terraform.yml` y `workflow-templates/infra-terraform.properties.json`.

## Capabilities

### New Capabilities

- `infra-terraform`: Reusable workflow que orquesta la validación, planificación y aplicación de infraestructura Terraform sobre AWS, con paso del tfplan entre jobs mediante artifacts, comentario automático del plan en Pull Requests, autenticación flexible en AWS (OIDC o credenciales estáticas) y gate de aprobación vía GitHub Environments.

### Modified Capabilities

## Impact

- **Inputs requeridos:** `terraform-version`, `working-directory`, `aws-region`, `environment-name`, `backend-config`, `aws-role-to-assume` (opcional, para OIDC)
- **Autenticación AWS:** soporta dos métodos de autenticación base:
  - OIDC: sin credenciales estáticas (recomendado)
  - Credenciales estáticas: secrets `aws-access-key-id` y `aws-secret-access-key`
  - En ambos casos, el input `aws-role-to-assume` permite asumir un IAM Role tras la autenticación
- **Outputs:** `plan-exit-code`
- **Dependencias externas:** `actions/checkout@v6`, `hashicorp/setup-terraform@v4`, `aws-actions/configure-aws-credentials@v6`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`, `actions/github-script@v9`
- **Repositorios beneficiados:** cualquier repositorio de la organización que gestione infraestructura con Terraform en AWS
- **Requisito de GitHub:** GitHub Environment configurado con protección de aprobación manual para el job `apply`
