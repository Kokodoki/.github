# build-docker

Reusable workflow que construye una imagen Docker y, de forma condicional, la publica en un repositorio de Amazon ECR. Separa la fase de construcción (siempre ejecutada) de la fase de publicación (controlada por un flag y protegida con un GitHub Environment).

## Cómo consumirlo

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  docker:
    uses: Kokodoki/.github/.github/workflows/build-docker.yml@<v1>
    with:
      aws-region: us-east-1
      ecr-repository: my-app
      environment-name: production
      enable-push: ${{ github.event_name == 'push' }}
      # image-tag: "1.0.0"
      # dockerfile: Dockerfile
      # context: .
      # build-args: |
      #   NODE_VERSION=20
      #   APP_ENV=production
      # aws-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    secrets:
      # Dejar vacíos si se usa OIDC puro (sin credenciales estáticas)
      # aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      # aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `aws-region` | string | **Sí** | — | Región de AWS donde está el repositorio de Amazon ECR |
| `ecr-repository` | string | **Sí** | — | Nombre del repositorio en Amazon ECR (sin el prefijo del registry) |
| `environment-name` | string | **Sí** | — | Nombre del GitHub Environment para el gate de aprobación en el push |
| `image-tag` | string | No | SHA del commit | Tag de la imagen Docker. Si se omite, se usa el SHA completo del commit |
| `dockerfile` | string | No | `Dockerfile` | Ruta al Dockerfile relativa al contexto de construcción |
| `context` | string | No | `.` | Contexto de construcción de Docker |
| `build-args` | string | No | `""` | Argumentos de construcción de Docker (formato `KEY=VALUE`, uno por línea) |
| `aws-role-to-assume` | string | No | `""` | ARN del IAM Role a asumir tras la autenticación base. Compatible con OIDC y credenciales estáticas |
| `enable-push` | boolean | No | `true` | Habilita el push de la imagen al ECR. Establecer en `false` para solo construir sin publicar |

## Secrets

| Secret | Requerido | Descripción |
|---|---|---|
| `aws-access-key-id` | No* | AWS Access Key ID para autenticación con credenciales estáticas |
| `aws-secret-access-key` | No* | AWS Secret Access Key para autenticación con credenciales estáticas |

*Al menos uno de los métodos de autenticación debe estar configurado cuando `enable-push` es `true`:
- **OIDC (recomendado):** configurar el IAM Identity Provider OIDC en la cuenta AWS. No se necesitan secrets de credenciales. Opcionalmente pasar `aws-role-to-assume`.
- **Credenciales estáticas:** pasar `aws-access-key-id` y `aws-secret-access-key`. Opcionalmente pasar `aws-role-to-assume` para asumir un role adicional.

## Outputs

| Output | Descripción |
|---|---|
| `image-uri` | URI completa de la imagen publicada en ECR (disponible solo si `enable-push` es `true`) |

## Permissions requeridos

El workflow requiere los siguientes permisos en el workflow consumidor:

```yaml
permissions:
  id-token: write  # Para autenticación OIDC con AWS
  contents: read   # Para checkout del repositorio
```

## Jobs

### `build`

Construye la imagen Docker y la almacena como artifact para ser consumida por el job `push`.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Establecer tag de imagen | Calcula el tag efectivo: usa `image-tag` si se proporcionó, de lo contrario usa el SHA del commit (`GITHUB_SHA`) |
| Set up Docker Buildx | Configura Docker Buildx para la construcción multi-plataforma |
| Build Docker image | Construye la imagen con `docker/build-push-action` sin publicarla. La exporta como tarball en `/tmp/image.tar` |
| Upload image artifact | Sube el tarball de la imagen como artifact `docker-image-<run_id>` con retención de 1 día |

### `push`

Publica la imagen construida en Amazon ECR. Solo se ejecuta si `enable-push` es `true` y requiere aprobación manual en el GitHub Environment configurado.

| Step | Descripción |
|---|---|
| Configure AWS Credentials | Autentica en AWS mediante OIDC o credenciales estáticas. Asume `aws-role-to-assume` si se proporciona |
| Login to Amazon ECR | Autentica Docker contra el registry de ECR en la región especificada |
| Download image artifact | Descarga el artifact `docker-image-<run_id>` generado en el job `build` |
| Load Docker image | Carga el tarball de la imagen en el daemon Docker local |
| Tag and push to ECR | Etiqueta la imagen con la URI completa del ECR y la publica. Expone la URI completa en el output `image-uri` |

## Configuración de GitHub Environment

El job `push` usa `environment: <environment-name>`. Para habilitar el gate de aprobación manual:

1. Ir a **Settings → Environments** en el repositorio consumidor.
2. Crear o editar el environment con el nombre pasado en `environment-name`.
3. En **Deployment protection rules**, habilitar **Required reviewers** y agregar los aprobadores.

Si no se desea gate de aprobación, se puede crear el environment sin protection rules. El job `push` seguirá siendo condicional al flag `enable-push`.

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

3. Asegurarse de que el IAM Role tenga permisos de ECR:
   - `ecr:GetAuthorizationToken`
   - `ecr:BatchCheckLayerAvailability`
   - `ecr:InitiateLayerUpload`
   - `ecr:UploadLayerPart`
   - `ecr:CompleteLayerUpload`
   - `ecr:PutImage`
