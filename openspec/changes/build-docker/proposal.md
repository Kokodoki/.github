## Why

Actualmente cada repositorio de la organización que necesita construir y publicar imágenes Docker debe duplicar los mismos steps de autenticación en ECR, build y push, generando inconsistencias y deuda de mantenimiento. Un reusable workflow centralizado estandariza este proceso para todos los repositorios consumidores.

## What Changes

- Nuevo reusable workflow `.github/workflows/build-docker.yml` que encapsula el ciclo completo de build y push de imágenes Docker hacia Amazon ECR.
- Documentación del workflow en `docs/build-docker.md` con descripción de inputs, outputs, secrets y ejemplo de uso.
- Workflow template en `workflow-templates/build-docker.yml` y `workflow-templates/build-docker.properties.json` para facilitar la adopción desde la UI de GitHub.

### Inputs del workflow

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `aws-region` | string | sí | — | Región de AWS donde está el repositorio ECR |
| `ecr-repository` | string | sí | — | Nombre del repositorio ECR (sin el prefijo de cuenta ni región) |
| `image-tag` | string | no | `${{ github.sha }}` | Tag de la imagen Docker a construir |
| `dockerfile` | string | no | `Dockerfile` | Ruta al Dockerfile relativa al contexto de build |
| `build-context` | string | no | `.` | Contexto de build de Docker |
| `build-args` | string | no | `""` | Build args adicionales en formato `KEY=VALUE` separados por saltos de línea |
| `aws-role-to-assume` | string | no | `""` | ARN del IAM Role a asumir (compatible con OIDC y credenciales estáticas) |

### Secrets del workflow

| Secret | Requerido | Descripción |
|---|---|---|
| `aws-access-key-id` | no | AWS Access Key ID para autenticación con credenciales estáticas |
| `aws-secret-access-key` | no | AWS Secret Access Key para autenticación con credenciales estáticas |

### Outputs del workflow

| Output | Descripción |
|---|---|
| `image-uri` | URI completa de la imagen publicada en ECR (incluye tag) |
| `image-digest` | Digest SHA256 de la imagen publicada |

## Capabilities

### New Capabilities

- `build-docker`: Construcción y publicación de imágenes Docker en Amazon ECR, con autenticación AWS mediante OIDC o credenciales estáticas, generando como outputs el URI completo y el digest de la imagen publicada.

### Modified Capabilities

## Impact

- **Repositorios beneficiados**: Todos los repositorios de la organización que construyan y publiquen imágenes Docker en Amazon ECR.
- **Dependencias externas**:
  - `actions/checkout@v4`
  - `aws-actions/configure-aws-credentials@v4`
  - `aws-actions/amazon-ecr-login@v2`
  - `docker/build-push-action@v6`
  - `docker/metadata-action@v5`
- **Secrets**: Requiere credenciales AWS a nivel de repositorio u organización (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) o un OIDC Identity Provider configurado.
- **Sin cambios breaking** en workflows existentes.
