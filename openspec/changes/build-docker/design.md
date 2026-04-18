## Context

El repositorio `.github` de la organización centraliza los reusable workflows de GitHub Actions. Actualmente existe el workflow `infra-terraform.yml` como referencia de patrón. El workflow `build-docker.yml` seguirá el mismo patrón: autenticación AWS (OIDC o credenciales estáticas), un único job con responsabilidad específica y outputs bien definidos.

El workflow publicará las imágenes en Amazon ECR, que es el registro de contenedores utilizado por la organización (alineado con la infraestructura AWS existente).

## Goals / Non-Goals

**Goals:**
- Encapsular el ciclo completo de build y push de imágenes Docker hacia Amazon ECR en un único workflow reutilizable.
- Soportar autenticación AWS mediante OIDC (sin credenciales) o credenciales estáticas, igual que `infra-terraform.yml`.
- Exponer outputs (`image-uri`, `image-digest`) para que los workflows consumidores puedan encadenar pasos de despliegue.
- Acompañar el workflow con documentación en `docs/build-docker.md` y template en `workflow-templates/`.

**Non-Goals:**
- Escaneo de vulnerabilidades de la imagen (puede ser un workflow `scan-docker.yml` separado en el futuro).
- Builds multi-plataforma (se puede agregar como extensión posterior).
- Soporte para registros distintos de ECR (Docker Hub, GHCR, ACR).
- Gestión del ciclo de vida de imágenes en ECR (limpieza de tags antiguos).

## Decisions

### Decisión 1: Un único job `build`

**Elección**: El workflow tendrá un único job llamado `build` que ejecuta autenticación, build y push de forma secuencial.

**Alternativa considerada**: Separar en jobs `authenticate`, `build` y `push`. Se descartó porque la autenticación y el push son atómicos — las credenciales de ECR expiran a los 12 horas y no tiene sentido persistirlas entre jobs. La atomicidad además simplifica el flujo de errores.

**Rationale**: Mismo patrón que seguirá `deploy-ecr.yml` en el futuro; mantiene la consistencia con un job por responsabilidad mayor (build+push es una operación indivisible en Docker).

### Decisión 2: Usar `docker/build-push-action` y `docker/metadata-action`

**Elección**: Utilizar las actions oficiales de Docker (`docker/build-push-action@v6`, `docker/metadata-action@v5`) sobre comandos `docker build` / `docker push` crudos.

**Alternativa considerada**: Invocar `docker build` y `docker push` directamente via `run`. Se descartó porque las actions de Docker abstraen la autenticación del registry, el caché de capas de BuildKit y la generación de metadatos OCI, reduciendo la complejidad del YAML y mejorando la reproducibilidad.

### Decisión 3: Autenticación AWS idéntica a `infra-terraform.yml`

**Elección**: Usar `aws-actions/configure-aws-credentials@v4` con los mismos inputs y secrets que el workflow de Terraform: OIDC puro, credenciales estáticas, o combinación con `aws-role-to-assume`.

**Rationale**: Consistencia con el patrón ya establecido en la organización; los repositorios consumidores ya conocen cómo configurar los secrets y el OIDC provider.

### Decisión 4: `image-tag` con default al SHA del commit

**Elección**: El input `image-tag` es opcional y por defecto usa `${{ github.sha }}`.

**Rationale**: El SHA del commit es inmutable, determinístico y trazable. Los workflows consumidores pueden sobreescribirlo con semver, branch name o cualquier convención de tagging.

### Estructura de archivos generados

```
.github/workflows/build-docker.yml          ← Reusable workflow
docs/build-docker.md                         ← Documentación
workflow-templates/build-docker.yml          ← Template para consumidores
workflow-templates/build-docker.properties.json ← Metadatos del template
```

### Estructura del job `build`

```
steps:
  1. Checkout
  2. Configure AWS Credentials (OIDC o estáticas)
  3. Login to Amazon ECR
  4. Build and push Docker image
```

### Inputs, outputs y secrets del workflow

**Inputs:**

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `aws-region` | string | sí | — | Región AWS del repositorio ECR |
| `ecr-repository` | string | sí | — | Nombre del repositorio ECR |
| `image-tag` | string | no | `${{ github.sha }}` | Tag de la imagen |
| `dockerfile` | string | no | `Dockerfile` | Ruta al Dockerfile |
| `build-context` | string | no | `.` | Contexto de build |
| `build-args` | string | no | `""` | Build args adicionales (`KEY=VALUE` por línea) |
| `aws-role-to-assume` | string | no | `""` | ARN del IAM Role a asumir |

**Secrets:**

| Secret | Requerido | Descripción |
|---|---|---|
| `aws-access-key-id` | no | AWS Access Key ID |
| `aws-secret-access-key` | no | AWS Secret Access Key |

**Outputs:**

| Output | Descripción |
|---|---|
| `image-uri` | URI completa (`<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`) |
| `image-digest` | Digest SHA256 de la imagen |

## Risks / Trade-offs

- **Credenciales ECR expiran a 12h** → Mitigation: El login a ECR se realiza dentro del mismo job; no se persisten credenciales entre jobs.
- **El input `build-args` como string multilínea puede ser frágil** → Mitigation: Se documenta el formato explícitamente; es el formato estándar que acepta `docker/build-push-action`.
- **Sin caché de capas por defecto** → Mitigation: Se puede agregar en el futuro como input opcional `cache-from`/`cache-to`; mantenerlo simple en v1.
- **ECR como único registro soportado** → Mitigation: Documentado explícitamente como Non-Goal; se puede crear `build-docker-ghcr.yml` o parametrizar si surge la necesidad.
