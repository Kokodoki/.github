# infra-terraform-azure

Reusable workflow que orquesta el ciclo de vida completo de Terraform sobre Azure: validación, planificación con gate de revisión en Pull Requests y aplicación con aprobación manual mediante GitHub Environments. La autenticación con Azure se realiza exclusivamente mediante OIDC (Workload Identity Federation), sin necesidad de credenciales estáticas.

## Prerequisitos

### Configuración de OIDC en Azure

Antes de usar este workflow, configura la autenticación OIDC en Azure:

1. Registrar una **aplicación en Entra ID** (Azure AD):
   - Ir a **Microsoft Entra ID → App registrations → New registration**.
   - Anotar el **Application (client) ID** y el **Directory (tenant) ID**.

2. Crear un **Federated Credential** en la aplicación registrada:
   - Ir a la aplicación → **Certificates & secrets → Federated credentials → Add credential**.
   - Seleccionar **GitHub Actions deploying Azure resources**.
   - Completar:
     - **Organization**: nombre de la organización GitHub.
     - **Repository**: nombre del repositorio que usará el workflow.
     - **Entity type**: `Branch`, `Pull request`, o `Environment` según el caso.
     - **GitHub branch / environment name**: rama o environment que disparará el workflow.

3. Asignar el **rol de Azure** necesario a la aplicación registrada sobre la suscripción o recurso objetivo (e.g., `Contributor`).

4. Almacenar los tres identificadores como variables en el repositorio consumidor:
   - `AZURE_CLIENT_ID` → Application (client) ID
   - `AZURE_TENANT_ID` → Directory (tenant) ID
   - `AZURE_SUBSCRIPTION_ID` → Subscription ID

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
    uses: Kokodoki/.github/.github/workflows/infra-terraform-azure.yml@v1
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
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos `.tf` |
| `azure-client-id` | string | **Sí** | — | Client ID de la aplicación registrada en Entra ID para autenticación OIDC |
| `azure-tenant-id` | string | **Sí** | — | Tenant ID de la organización en Entra ID |
| `azure-subscription-id` | string | **Sí** | — | Subscription ID de Azure donde se despliega la infraestructura |
| `environment-name` | string | **Sí** | — | Nombre del GitHub Environment para el gate de aprobación en `apply` |
| `backend-config` | string | No | `""` | Flags de configuración del backend de Terraform para Azure (e.g. `-backend-config=resource_group_name=rg-tfstate`) |
| `enable-apply` | boolean | No | `true` | Habilita el job `apply`. Establecer en `false` para omitir el apply (e.g. en Pull Requests) |

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

Genera el plan de cambios y lo publica para revisión. Requiere acceso al backend remoto y autenticación OIDC con Azure.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Azure Login (OIDC) | Autentica en Azure mediante OIDC usando `azure/login@v2` con `azure-client-id`, `azure-tenant-id` y `azure-subscription-id` |
| Terraform Init | Inicializa con backend real usando los flags de `backend-config` |
| Terraform Plan | Ejecuta `terraform plan -detailed-exitcode -out=tfplan`. El exit code se captura en el output `plan-exit-code` |
| Upload tfplan artifact | Sube el binario `tfplan` y el output del plan como artifact `tfplan-<run_id>` con retención de 7 días |
| Comentar plan en PR | Si el evento es `pull_request`, publica o actualiza un comentario en el PR con el output del plan. Se omite en otros eventos |

### `apply`

Descarga el plan aprobado y lo aplica. Requiere aprobación manual en el GitHub Environment antes de ejecutar. Solo se ejecuta si `enable-apply` es `true`.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Azure Login (OIDC) | Autentica en Azure mediante OIDC con las mismas credenciales que `plan` |
| Terraform Init | Inicializa con backend real para acceder al estado remoto |
| Download tfplan artifact | Descarga el artifact `tfplan-<run_id>` generado en el job `plan` |
| Terraform Apply | Ejecuta `terraform apply -auto-approve tfplan` con el plan exacto que fue revisado y aprobado |

## Configuración de GitHub Environment

El job `apply` usa `environment: <environment-name>`. Para habilitar el gate de aprobación manual:

1. Ir a **Settings → Environments** en el repositorio consumidor.
2. Crear o editar el environment con el nombre pasado en `environment-name`.
3. En **Deployment protection rules**, habilitar **Required reviewers** y agregar los aprobadores.
