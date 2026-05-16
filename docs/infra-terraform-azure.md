# infra-terraform-azure

Reusable workflow que orquesta el ciclo de vida completo de Terraform sobre Azure: validación, planificación con gate de revisión en Pull Requests y aplicación con aprobación manual mediante GitHub Environments. Usa OIDC (Workload Identity Federation) para autenticarse en Azure sin necesidad de secretos de larga duración.

## Cómo consumirlo

```yaml
name: Infraestructura Azure

on:
  push:
    branches: [$default-branch]
  pull_request:
    branches: [$default-branch]

jobs:
  terraform:
    uses: kvncont/.github/.github/workflows/infra-terraform-azure.yml@v1
    with:
      azure-client-id: ${{ vars.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ vars.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      environment-name: production
      working-directory: infra/
      terraform-version: "1.9.0"
      backend-config: >-
        -backend-config=resource_group_name=rg-tfstate
        -backend-config=storage_account_name=sttfstate
        -backend-config=container_name=tfstate
        -backend-config=key=prod/terraform.tfstate
      enable-apply: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      # Dejar vacío si se usa OIDC (recomendado)
      azure-client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos `.tf` |
| `azure-client-id` | string | **Sí** | — | Client ID del App Registration de Azure para autenticación OIDC |
| `azure-tenant-id` | string | **Sí** | — | Tenant ID del directorio de Azure Active Directory |
| `azure-subscription-id` | string | **Sí** | — | Subscription ID de Azure donde se despliega la infraestructura |
| `environment-name` | string | **Sí** | — | Nombre del GitHub Environment para el gate de aprobación en `apply` |
| `backend-config` | string | No | `""` | Flags de configuración del backend de Terraform (e.g. `-backend-config=resource_group_name=rg-tfstate`) |
| `enable-apply` | boolean | No | `true` | Habilita el job `apply`. Establecer en `false` para omitir el apply (e.g. en Pull Requests) |

## Secrets

| Secret | Requerido | Descripción |
|---|---|---|
| `azure-client-secret` | No* | Client Secret del App Registration para autenticación con Service Principal clásico |

*Se recomienda usar OIDC. El secret `azure-client-secret` solo es necesario si el App Registration **no** tiene Federated Credentials configuradas.

## Outputs

| Output | Descripción |
|---|---|
| `plan-exit-code` | Código de salida de `terraform plan` (`0` = sin cambios, `2` = con cambios) |

## Permissions requeridos

El workflow requiere los siguientes permisos en el workflow consumidor:

```yaml
permissions:
  id-token: write      # Para autenticación OIDC con Azure
  contents: read       # Para checkout del repositorio
  pull-requests: write # Para comentar el plan en PRs
```

## Jobs

### `validate`

Ejecuta validaciones estáticas sin necesidad de backend remoto ni credenciales de Azure.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Terraform Format | Verifica el formato con `terraform fmt -check -recursive`. Falla si hay archivos mal formateados |
| Terraform Init (sin backend) | Inicializa el directorio de trabajo con `terraform init -backend=false` para resolver providers sin conectar al backend |
| Terraform Validate | Valida la sintaxis y coherencia de los archivos `.tf` con `terraform validate` |

### `plan`

Genera el plan de cambios y lo publica para revisión. Requiere acceso al backend remoto y credenciales de Azure.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Azure Login | Autentica en Azure mediante OIDC o Service Principal. Usa `azure/login@v2` |
| Terraform Init | Inicializa con backend real usando los flags de `backend-config` |
| Terraform Plan | Ejecuta `terraform plan -detailed-exitcode -out=tfplan`. El exit code se captura en el output `plan-exit-code` |
| Upload tfplan artifact | Sube el binario `tfplan` y el output del plan como artifact `tfplan-<run_id>` con retención de 7 días |
| Comentar plan en PR | Si el evento es `pull_request`, publica o actualiza un comentario en el PR con el output del plan. Se omite en otros eventos |

### `apply`

Descarga el plan aprobado y lo aplica. Requiere aprobación manual en el GitHub Environment antes de ejecutar. Se omite si `enable-apply` es `false`.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Azure Login | Autentica en Azure con las mismas credenciales que `plan` |
| Terraform Init | Inicializa con backend real para acceder al estado remoto |
| Download tfplan artifact | Descarga el artifact `tfplan-<run_id>` generado en el job `plan` |
| Terraform Apply | Ejecuta `terraform apply -auto-approve tfplan` con el plan exacto que fue revisado y aprobado |

## Configuración de GitHub Environment

El job `apply` usa `environment: <environment-name>`. Para habilitar el gate de aprobación manual:

1. Ir a **Settings → Environments** en el repositorio consumidor.
2. Crear o editar el environment con el nombre pasado en `environment-name`.
3. En **Deployment protection rules**, habilitar **Required reviewers** y agregar los aprobadores.

## Configuración de OIDC en Azure

Para usar autenticación OIDC (recomendado), configurar en Azure:

1. **Crear un App Registration** en Azure Active Directory (o usar uno existente).
2. En el App Registration, ir a **Certificates & secrets → Federated credentials** y agregar una nueva credencial:
   - **Scenario:** GitHub Actions deploying Azure resources
   - **Organization:** nombre de la organización en GitHub
   - **Repository:** nombre del repositorio
   - **Entity type:** Branch, Tag, Pull request o Environment (según el caso de uso)
   - **GitHub branch / tag name:** nombre de la rama o `*` para todas
3. Asignar al App Registration el rol de Azure RBAC necesario (e.g. `Contributor`) en la Subscription o Resource Group objetivo.
4. En el repositorio de GitHub, configurar los siguientes secrets/variables:
   - `AZURE_CLIENT_ID` → Application (client) ID del App Registration
   - `AZURE_TENANT_ID` → Directory (tenant) ID
   - `AZURE_SUBSCRIPTION_ID` → Subscription ID de Azure

### Ejemplo de llamada con OIDC

```yaml
jobs:
  terraform:
    uses: kvncont/.github/.github/workflows/infra-terraform-azure.yml@v1
    with:
      azure-client-id: ${{ vars.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ vars.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      environment-name: production
    # No se pasa azure-client-secret → autenticación OIDC pura
```

## Configuración del backend de Terraform para Azure

El backend recomendado para Azure es `azurerm`. Ejemplo de configuración en el repositorio consumidor:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstateprod"
    container_name       = "tfstate"
    key                  = "prod/terraform.tfstate"
  }
}
```

Pasar los valores dinámicos via `backend-config`:

```yaml
with:
  backend-config: >-
    -backend-config=resource_group_name=rg-tfstate
    -backend-config=storage_account_name=sttfstateprod
    -backend-config=container_name=tfstate
    -backend-config=key=prod/terraform.tfstate
```
