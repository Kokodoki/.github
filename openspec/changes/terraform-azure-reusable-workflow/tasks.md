## 1. Reusable Workflow

- [x] 1.1 Crear `.github/workflows/infra-terraform-azure.yml` con trigger `on: workflow_call`, inputs `terraform-version`, `working-directory`, `environment-name`, `backend-config` y `enable-apply`; secrets `azure-client-id`, `azure-tenant-id`, `azure-subscription-id` y `azure-client-secret`; y output `plan-exit-code`
- [x] 1.2 Implementar el job `validate` con steps: checkout, setup-terraform, terraform fmt, terraform init -backend=false, terraform validate
- [x] 1.3 Implementar el job `plan` con steps: checkout, setup-terraform, azure/login@v2 (OIDC o client secret), terraform init con `backend-config`, terraform plan -detailed-exitcode -out=tfplan, upload artifact `tfplan-<run_id>`, comentar plan en PR (solo si evento es pull_request)
- [x] 1.4 Implementar el job `apply` con steps: checkout, setup-terraform, azure/login@v2, terraform init, download artifact `tfplan-<run_id>`, terraform apply -auto-approve tfplan; protegido por `environment: ${{ inputs.environment-name }}` y condicionado por `inputs.enable-apply`

## 2. Documentación

- [x] 2.1 Crear `docs/infra-terraform-azure.md` con propósito del workflow, prerequisitos de configuración OIDC en Entra ID, ejemplo de uso completo con `uses:` apuntando al tag flotante, tabla de inputs/outputs/secrets y descripción de cada job y sus steps

## 3. Workflow Template

- [x] 3.1 Crear `workflow-templates/infra-terraform-azure.yml` con el template de consumo que usa `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1`, triggers en `$default-branch` y valores de ejemplo para todos los inputs y secrets
- [x] 3.2 Crear `workflow-templates/infra-terraform-azure.properties.json` con `name`, `description`, `iconName` y `categories` del template
