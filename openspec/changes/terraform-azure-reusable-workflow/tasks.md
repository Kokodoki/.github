## 1. Reusable Workflow

- [ ] 1.1 Crear `.github/workflows/infra-terraform-azure.yml` con `on: workflow_call` y la definición de inputs (`terraform-version`, `working-directory`, `environment-name`, `enable-apply`, `backend-resource-group-name`, `backend-storage-account-name`, `backend-container-name`, `backend-key`, `backend-config`), secrets (`azure-client-id`, `azure-client-secret`, `azure-tenant-id`, `azure-subscription-id`) y output `plan-exit-code`
- [ ] 1.2 Implementar el job `validate` con los steps: checkout, setup-terraform, `terraform fmt -check -recursive`, `terraform init -backend=false` y `terraform validate`
- [ ] 1.3 Implementar el job `plan` con los steps: checkout, setup-terraform, `azure/login@v2`, `terraform init` con configuración de backend Azure Blob Storage, `terraform plan -detailed-exitcode -out=tfplan`, upload del artifact `tfplan-<run_id>` y comentario del plan en PRs vía `actions/github-script@v9`
- [ ] 1.4 Implementar el job `apply` con los steps: checkout, setup-terraform, `azure/login@v2`, `terraform init`, download del artifact `tfplan-<run_id>` y `terraform apply tfplan`; configurar `environment: ${{ inputs.environment-name }}` y `if: inputs.enable-apply`
- [ ] 1.5 Configurar `permissions: id-token: write, contents: read, pull-requests: write` a nivel de workflow

## 2. Documentación

- [ ] 2.1 Crear `docs/infra-terraform-azure.md` con: propósito del workflow, prerrequisitos (recursos de Azure para backend, Federated Identity Credential o Service Principal), ejemplo completo de uso con OIDC y con Service Principal, tabla de inputs con tipo/requerido/default/descripción, tabla de secrets, tabla de outputs y descripción de cada job y sus steps

## 3. Workflow Template

- [ ] 3.1 Crear `workflow-templates/infra-terraform-azure.yml` con los triggers `on.push.branches: [$default-branch]` y `on.pull_request.branches: [$default-branch]`, y el job llamador usando `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1` con los inputs y secrets requeridos
- [ ] 3.2 Crear `workflow-templates/infra-terraform-azure.properties.json` con `name`, `description`, `iconName` y `categories` según la convención de GitHub workflow templates
