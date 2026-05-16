## 1. Reusable Workflow

- [ ] 1.1 Crear `.github/workflows/infra-terraform-azure.yml` con el trigger `on: workflow_call`, inputs (`terraform-version`, `working-directory`, `azure-subscription-id`, `environment-name`, `backend-resource-group-name`, `backend-storage-account-name`, `backend-container-name`, `backend-key`, `enable-apply`), secrets (`azure-client-id`, `azure-tenant-id`, `azure-subscription-id`, `azure-client-secret`) y output `plan-exit-code`
- [ ] 1.2 Implementar job `validate` con steps: checkout, setup-terraform, terraform fmt, terraform init -backend=false, terraform validate
- [ ] 1.3 Implementar job `plan` con steps: checkout, setup-terraform, azure/login, terraform init (azurerm backend), terraform plan -out=tfplan, upload artifact, comentario en PR
- [ ] 1.4 Implementar job `apply` con steps: checkout, setup-terraform, azure/login, terraform init, download artifact, terraform apply tfplan; condicionado por `enable-apply` y `environment-name`

## 2. Documentación

- [ ] 2.1 Crear `docs/infra-terraform-azure.md` con propósito del workflow, ejemplo de uso (llamada desde repositorio consumidor), tabla de inputs con nombre/tipo/requerido/descripción, tabla de secrets con nombre/requerido/descripción, tabla de outputs y descripción de cada job y sus steps

## 3. Workflow Template

- [ ] 3.1 Crear `workflow-templates/infra-terraform-azure.yml` con triggers `on.push.branches: [$default-branch]` y `on.pull_request.branches: [$default-branch]`, e invocación al reusable workflow con `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1` y los inputs/secrets requeridos
- [ ] 3.2 Crear `workflow-templates/infra-terraform-azure.properties.json` con `name`, `description` y `iconName` del template
