# infra-terraform-azure

Reusable workflow que orquesta el ciclo de vida completo de Terraform sobre Azure: validación, planificación con gate de revisión en Pull Requests y aplicación con aprobación manual mediante GitHub Environments. Autentica contra Azure mediante OIDC (Federated Identity Credentials) o Service Principal con client secret usando `azure/login@v2`, y almacena el estado remoto en Azure Storage Account.

## Cómo consumirlo

```yaml
name: Infraestructura Azure

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    uses: kvncont/.github/.github/workflows/infra-terraform-azure.yml@v1
    with:
      environment-name: production
      backend-resource-group-name: rg-terraform-state
      backend-storage-account-name: mystorageaccount
      backend-container-name: tfstate
      backend-key: prod/terraform.tfstate
      # working-directory: infra/
      # terraform-version: "1.9.0"
      # enable-apply: true
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      # azure-client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}  # Solo para Service Principal; omitir para OIDC
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos `.tf` |
| `environment-name` | string | **Sí** | — | Nombre del GitHub Environment para el gate de aprobación en `apply` |
| `backend-resource-group-name` | string | **Sí** | — | Nombre del Resource Group de Azure donde está el Storage Account del backend |
| `backend-storage-account-name` | string | **Sí** | — | Nombre del Storage Account de Azure para el backend de Terraform |
| `backend-container-name` | string | **Sí** | — | Nombre del container del Storage Account para el backend de Terraform |
| `backend-key` | string | **Sí** | — | Nombre del blob (clave) donde se almacena el estado de Terraform |
| `enable-apply` | boolean | No | `true` | Habilita el job `apply`. Establecer en `false` para omitir el apply (e.g. en Pull Requests) |

## Secrets

| Secret | Requerido | Descripción |
|---|---|---|
| `azure-client-id` | **Sí** | Client ID del App Registration de Azure AD (requerido para OIDC y Service Principal) |
| `azure-tenant-id` | **Sí** | Tenant ID del directorio de Azure AD |
| `azure-subscription-id` | **Sí** | Subscription ID de Azure donde se despliega la infraestructura |
| `azure-client-secret` | No* | Client Secret del App Registration. Solo necesario para Service Principal; omitir para OIDC |

*Métodos de autenticación soportados:
- **OIDC (recomendado):** configurar una Federated Identity Credential en el App Registration de Azure AD. No se necesita `azure-client-secret`.
- **Service Principal con client secret:** pasar `azure-client-secret` junto a los tres secrets requeridos.

## Outputs

| Output | Descripción |
|---|---|
| `plan-exit-code` | Código de salida de `terraform plan` (`0` = sin cambios, `2` = con cambios) |

## Permissions requeridos

El workflow requiere los siguientes permisos en el workflow consumidor:

```yaml
permissions:
  id-token: write      # Para autenticación OIDC con Azure AD
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
| Azure Login | Autentica en Azure mediante OIDC o Service Principal usando `azure/login@v2` |
| Terraform Init | Inicializa con backend `azurerm` usando los cuatro inputs de backend |
| Terraform Plan | Ejecuta `terraform plan -detailed-exitcode -out=tfplan`. El exit code se captura en el output `plan-exit-code` |
| Upload tfplan artifact | Sube el binario `tfplan` y el output del plan como artifact `tfplan-<run_id>` con retención de 7 días |
| Comentar plan en PR | Si el evento es `pull_request`, publica o actualiza un comentario en el PR con el output del plan. Se omite en otros eventos |

### `apply`

Descarga el plan aprobado y lo aplica. Requiere aprobación manual en el GitHub Environment antes de ejecutar.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Azure Login | Autentica en Azure con las mismas credenciales que `plan` |
| Terraform Init | Inicializa con backend `azurerm` para acceder al estado remoto |
| Download tfplan artifact | Descarga el artifact `tfplan-<run_id>` generado en el job `plan` |
| Terraform Apply | Ejecuta `terraform apply -auto-approve tfplan` con el plan exacto que fue revisado y aprobado |

## Configuración de GitHub Environment

El job `apply` usa `environment: <environment-name>`. Para habilitar el gate de aprobación manual:

1. Ir a **Settings → Environments** en el repositorio consumidor.
2. Crear o editar el environment con el nombre pasado en `environment-name`.
3. En **Deployment protection rules**, habilitar **Required reviewers** y agregar los aprobadores.

## Configuración de OIDC en Azure

Para usar autenticación OIDC, configurar en Azure AD:

1. Crear o usar un **App Registration** en Azure Active Directory.
2. En el App Registration, ir a **Certificates & secrets → Federated credentials → Add credential**.
3. Seleccionar **GitHub Actions deploying Azure resources** y configurar:
   - **Organization**: nombre de la organización o usuario de GitHub
   - **Repository**: nombre del repositorio
   - **Entity type**: `Branch`, `Tag` o `Environment` según el caso
   - **GitHub branch / tag / environment name**: valor correspondiente (e.g. `main`)
4. Asignar al App Registration los permisos de Azure (IAM) necesarios sobre la suscripción o resource group objetivo.
5. Configurar los secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` y `AZURE_SUBSCRIPTION_ID` en el repositorio o la organización de GitHub.
