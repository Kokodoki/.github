# infra-terraform-azure

Reusable workflow que orquesta el ciclo de vida completo de Terraform sobre Azure: validación, planificación con gate de revisión en Pull Requests y aplicación con aprobación manual mediante GitHub Environments. Soporta autenticación OIDC (Workload Identity Federation) y client secret.

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
      working-directory: infra/
      terraform-version: "1.9.0"
      backend-config: >-
        -backend-config=resource_group_name=my-tfstate-rg
        -backend-config=storage_account_name=mytfstate
        -backend-config=container_name=tfstate
        -backend-config=key=prod/terraform.tfstate
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      # Dejar sin definir si se usa OIDC (Workload Identity Federation)
      # azure-client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos `.tf` |
| `environment-name` | string | **Sí** | — | Nombre del GitHub Environment para el gate de aprobación en `apply` |
| `backend-config` | string | No | `""` | Flags de configuración del backend de Terraform (e.g. `-backend-config=resource_group_name=my-rg`) |
| `enable-apply` | boolean | No | `true` | Habilita el job `apply`. Establecer en `false` para omitir el apply (e.g. en Pull Requests) |

## Secrets

| Secret | Requerido | Descripción |
|---|---|---|
| `azure-client-id` | **Sí** | Client ID de la App Registration en Entra ID |
| `azure-tenant-id` | **Sí** | Tenant ID del directorio de Azure Active Directory |
| `azure-subscription-id` | **Sí** | Subscription ID de Azure donde se despliega la infraestructura |
| `azure-client-secret` | No* | Client Secret de la App Registration. Si se omite, se usa autenticación OIDC |

*Si `azure-client-secret` **no se proporciona**, el workflow usa **OIDC (Workload Identity Federation)** — el método recomendado. Si se proporciona, usa autenticación con **client secret**.

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
| Terraform Init (sin backend) | Inicializa con `terraform init -backend=false` para resolver providers sin conectar al backend |
| Terraform Validate | Valida la sintaxis y coherencia de los archivos `.tf` con `terraform validate` |

### `plan`

Genera el plan de cambios y lo publica para revisión. Requiere acceso al backend remoto y credenciales de Azure.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Configure Azure Credentials | Autentica en Azure mediante OIDC o client secret usando `azure/login@v2`. Las variables `ARM_*` se configuran a nivel de job para el provider `azurerm` de Terraform |
| Terraform Init | Inicializa con backend real usando los flags de `backend-config` |
| Terraform Plan | Ejecuta `terraform plan -detailed-exitcode -out=tfplan`. El exit code se captura en el output `plan-exit-code` |
| Upload tfplan artifact | Sube el binario `tfplan` y el output del plan como artifact `tfplan-<run_id>` con retención de 7 días |
| Comentar plan en PR | Si el evento es `pull_request`, publica o actualiza un comentario en el PR con el output del plan. Se omite en otros eventos |

### `apply`

Descarga el plan aprobado y lo aplica. Requiere aprobación manual en el GitHub Environment antes de ejecutar.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Configure Azure Credentials | Autentica en Azure con las mismas credenciales que `plan` |
| Terraform Init | Inicializa con backend real para acceder al estado remoto |
| Download tfplan artifact | Descarga el artifact `tfplan-<run_id>` generado en el job `plan` |
| Terraform Apply | Ejecuta `terraform apply -auto-approve tfplan` con el plan exacto que fue revisado y aprobado |

## Configuración de GitHub Environment

El job `apply` usa `environment: <environment-name>`. Para habilitar el gate de aprobación manual:

1. Ir a **Settings → Environments** en el repositorio consumidor.
2. Crear o editar el environment con el nombre pasado en `environment-name`.
3. En **Deployment protection rules**, habilitar **Required reviewers** y agregar los aprobadores.

## Configuración de OIDC en Azure (recomendado)

Para usar autenticación OIDC (Workload Identity Federation) sin client secret:

1. **Crear o usar una App Registration** en Entra ID (Azure Active Directory).

2. **Agregar una Federated Credential** en la App Registration:
   - Ve a la App Registration → **Certificates & secrets** → **Federated credentials** → **Add credential**.
   - Selecciona el escenario **GitHub Actions deploying Azure resources**.
   - Completa los campos:
     - **Organization**: nombre de la organización en GitHub (e.g. `mi-org`)
     - **Repository**: nombre del repositorio (e.g. `mi-repo`)
     - **Entity type**: `Branch`, `Environment`, `Pull Request` o `Tag` según el caso de uso
     - **GitHub branch/environment/tag name**: e.g. `main` o el environment name

3. **Asignar el role** correspondiente al Service Principal en la suscripción o resource group de Azure (e.g. `Contributor`).

4. **Guardar los valores** como secrets en el repositorio o a nivel de organización:
   - `AZURE_CLIENT_ID` → Application (client) ID de la App Registration
   - `AZURE_TENANT_ID` → Directory (tenant) ID
   - `AZURE_SUBSCRIPTION_ID` → Subscription ID de Azure

## Configuración de autenticación con Client Secret (alternativa)

Si no es posible usar OIDC:

1. En la App Registration, ir a **Certificates & secrets** → **Client secrets** → **New client secret**.
2. Copiar el valor generado.
3. Guardar el valor como secret `AZURE_CLIENT_SECRET` en el repositorio.
4. Pasar `azure-client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}` al invocar el workflow.
