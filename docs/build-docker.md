# build-docker

Reusable workflow que encapsula el ciclo completo de construcción y publicación de imágenes Docker en Amazon ECR, con autenticación en AWS mediante OIDC o credenciales estáticas.

## Cómo consumirlo

```yaml
name: Build

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-docker:
    uses: Kokodoki/.github/.github/workflows/build-docker.yml@v1
    with:
      aws-region: us-east-1
      ecr-repository: mi-app
      image-tag: ${{ github.sha }}
      # dockerfile: Dockerfile
      # build-context: .
      # build-args: |
      #   NODE_ENV=production
      #   APP_VERSION=1.0.0
      # aws-role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    secrets:
      # Dejar vacíos si se usa OIDC (aws-role-to-assume sin credenciales estáticas)
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Encadenar con un job de despliegue

```yaml
jobs:
  build-docker:
    uses: Kokodoki/.github/.github/workflows/build-docker.yml@v1
    with:
      aws-region: us-east-1
      ecr-repository: mi-app

  deploy:
    needs: build-docker
    runs-on: ubuntu-latest
    steps:
      - name: Deploy image
        run: |
          echo "Deploying ${{ needs.build-docker.outputs.image-uri }}"
          echo "Digest: ${{ needs.build-docker.outputs.image-digest }}"
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `aws-region` | string | **Sí** | — | Región de AWS donde está el repositorio ECR |
| `ecr-repository` | string | **Sí** | — | Nombre del repositorio ECR (sin el prefijo de cuenta ni región) |
| `image-tag` | string | No | `${{ github.sha }}` | Tag de la imagen Docker a construir |
| `dockerfile` | string | No | `Dockerfile` | Ruta al Dockerfile relativa a la raíz del repositorio |
| `build-context` | string | No | `.` | Contexto de build de Docker |
| `build-args` | string | No | `""` | Build args adicionales en formato `KEY=VALUE` separados por saltos de línea |
| `aws-role-to-assume` | string | No | `""` | ARN del IAM Role a asumir tras la autenticación base. Compatible con OIDC y credenciales estáticas |

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
| `image-uri` | URI completa de la imagen publicada en ECR (formato: `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`) |
| `image-digest` | Digest SHA256 de la imagen publicada |

## Permissions requeridos

El workflow requiere los siguientes permisos en el workflow consumidor:

```yaml
permissions:
  id-token: write   # Para autenticación OIDC con AWS
  contents: read    # Para checkout del repositorio
```

## Jobs

### `build`

Ejecuta el ciclo completo de autenticación en AWS, login en ECR, y construcción y publicación de la imagen Docker.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio |
| Configure AWS Credentials | Autentica en AWS mediante OIDC o credenciales estáticas. Asume `aws-role-to-assume` si se proporciona |
| Login to Amazon ECR | Obtiene un token de autenticación válido para el registro ECR de la cuenta AWS |
| Set up Docker Buildx | Configura Docker Buildx para soporte de builds avanzados |
| Build and push Docker image | Construye la imagen desde el `dockerfile` y `build-context` indicados, y la publica en ECR con el tag `image-tag` |
| Set image URI output | Compone el URI completo de la imagen y lo expone como output `image-uri` |

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

3. Adjuntar al role las políticas necesarias para interactuar con ECR:
   - `AmazonEC2ContainerRegistryPowerUser` (para push) o una política personalizada de mínimo privilegio.
