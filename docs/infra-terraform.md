# infra-terraform

Reusable workflow que orquesta el ciclo de vida completo de Terraform sobre AWS: validación, planificación con gate de revisión en Pull Requests y aplicación con aprobación manual mediante GitHub Environments.

## Cómo consumirlo

```yaml
name: Infraestructura

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    uses: kvncont/.github/.github/workflows/infra-terraform.yml<@v1>
    with:
      aws-region: us-east-1
      environment-name: production
      working-directory: infra/
      terraform-version: "1.9.0"
      backend-config: -backend-config=bucket=my-tfstate -backend-config=key=prod/terraform.tfstate
      aws-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    secrets:
      # Dejar vacíos si se usa OIDC (aws-role-to-assume sin credenciales estáticas)
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos `.tf` |
| `aws-region` | string | **Sí** | — | Región de AWS donde se despliega la infraestructura |
| `environment-name` | string | **Sí** | — | Nombre del GitHub Environment para el gate de aprobación en `apply` |
| `backend-config` | string | No | `""` | Flags de configuración del backend de Terraform (e.g. `-backend-config=bucket=my-bucket -backend-config=key=prod/terraform.tfstate`) |
| `aws-role-to-assume` | string | No | `""` | ARN del IAM Role a asumir tras la autenticación base. Compatible con OIDC y credenciales estáticas |
| `enable-apply` | boolean | No | `true` | Habilita el job `apply`. Establecer en `false` para omitir el apply (e.g. en Pull Requests) |

## Secrets

| Secret | Requerido | Descripción |
|---|---|---|
| `aws-access-key-id` | No* | AWS Access Key ID para autenticación con credenciales estáticas |
| `aws-secret-access-key` | No* | AWS Secret Access Key para autenticación con credenciales estáticas |

*Al menos uno de los métodos de autenticación debe estar configurado:
- **OIDC (recomendado):** configurar el IAM Identity Provider OIDC en la cuenta AWS. No se necesitan secrets de credenciales. Opcionalmente pasar `aws-role-to-assume`.
- **Credenciales estáticas:** pasar `aws-access-key-id` y `aws-secret-access-key`. Opcionalmente pasar `aws-role-to-assume` para asumir un role adicional.

## Outputs

| Output | Descripción |
|---|---|
| `plan-exit-code` | Código de salida de `terraform plan` (`0` = sin cambios, `2` = con cambios) |

## Permissions requeridos

El workflow requiere los siguientes permisos en el workflow consumidor:

```yaml
permissions:
  id-token: write      # Para autenticación OIDC con AWS
  contents: read       # Para checkout del repositorio
  pull-requests: write # Para comentar el plan en PRs
```

## Jobs

### `validate`

Ejecuta validaciones estáticas sin necesidad de backend remoto ni credenciales de AWS.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Terraform Format | Verifica el formato con `terraform fmt -check -recursive`. Falla si hay archivos mal formateados |
| Terraform Init (sin backend) | Inicializa el directorio de trabajo con `terraform init -backend=false` para resolver providers sin conectar al backend |
| Terraform Validate | Valida la sintaxis y coherencia de los archivos `.tf` con `terraform validate` |

### `plan`

Genera el plan de cambios y lo publica para revisión. Requiere acceso al backend remoto y credenciales de AWS.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Setup Terraform | Instala la versión especificada de Terraform |
| Configure AWS Credentials | Autentica en AWS mediante OIDC o credenciales estáticas. Asume `aws-role-to-assume` si se proporciona |
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
| Configure AWS Credentials | Autentica en AWS con las mismas credenciales que `plan` |
| Terraform Init | Inicializa con backend real para acceder al estado remoto |
| Download tfplan artifact | Descarga el artifact `tfplan-<run_id>` generado en el job `plan` |
| Terraform Apply | Ejecuta `terraform apply -auto-approve tfplan` con el plan exacto que fue revisado y aprobado |

## Configuración de GitHub Environment

El job `apply` usa `environment: <environment-name>`. Para habilitar el gate de aprobación manual:

1. Ir a **Settings → Environments** en el repositorio consumidor.
2. Crear o editar el environment con el nombre pasado en `environment-name`.
3. En **Deployment protection rules**, habilitar **Required reviewers** y agregar los aprobadores.

## Configuración de OIDC en AWS

Para usar autenticación OIDC, configurar en la cuenta AWS:

1. Crear un **IAM Identity Provider** con:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
2. Crear un **IAM Role** con trust policy que permita al repositorio específico asumir el role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<org>/<repo>:*"
        }
      }
    }
  ]
}
```
