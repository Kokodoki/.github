## 1. Reusable Workflow

- [x] 1.1 Crear `.github/workflows/infra-terraform-azure.yml` con los inputs `terraform-version`, `working-directory`, `azure-client-id`, `azure-tenant-id`, `azure-subscription-id`, `environment-name`, `backend-config` y `enable-apply`
- [x] 1.2 Implementar el job `validate` con los steps: checkout, setup-terraform, terraform fmt, terraform init sin backend y terraform validate
- [x] 1.3 Implementar el job `plan` con los steps: checkout, setup-terraform, azure/login (OIDC), terraform init con backend, terraform plan con `-detailed-exitcode -out=tfplan`, upload del artifact `tfplan-<run_id>` y comentario del plan en Pull Requests
- [x] 1.4 Implementar el job `apply` condicionado por `enable-apply`, con el GitHub Environment `environment-name`, y los steps: checkout, setup-terraform, azure/login (OIDC), terraform init con backend, download del artifact `tfplan-<run_id>` y terraform apply

## 2. Documentación

- [x] 2.1 Crear `docs/infra-terraform-azure.md` con propósito del workflow, tabla de inputs/outputs, descripción de cada job y sus steps, prerequisitos de configuración OIDC en Azure y ejemplo de uso completo

## 3. Workflow Template

- [x] 3.1 Crear `workflow-templates/infra-terraform-azure.yml` con la referencia al reusable workflow usando tag flotante mayor (`@v1`) y los inputs necesarios pre-rellenados
- [x] 3.2 Crear `workflow-templates/infra-terraform-azure.properties.json` con los metadatos del template (name, description, iconName, categories)
