## Context

La organización carece de un estándar centralizado para ejecutar pipelines de Terraform en AWS. Cada repositorio define sus propios jobs con distintas versiones de Terraform, distintas formas de autenticarse en AWS y sin un gate de aprobación consistente antes del apply. Esto genera deuda técnica, riesgo operacional y dificultad para auditar cambios de infraestructura.

El workflow `infra-terraform.yml` se aloja en el repositorio especial `.github` y es consumido por cualquier repositorio de la organización que gestione infraestructura con Terraform sobre AWS.

## Goals / Non-Goals

**Goals:**

- Estandarizar los tres jobs del ciclo de vida de Terraform: `validate`, `plan` y `apply`.
- Pasar el `tfplan` binario del job `plan` al job `apply` mediante GitHub Actions Artifacts, garantizando que el plan aprobado es exactamente el que se aplica.
- Requerir aprobación manual mediante GitHub Environment antes del `apply`.
- Autenticarse en AWS de forma flexible: OIDC (recomendado) o credenciales estáticas (`aws-access-key-id` + `aws-secret-access-key`).
- Comentar el output de `terraform plan` en el Pull Request cuando el evento disparador es `pull_request`.
- Exponer inputs, outputs y secrets en kebab-case.
- Acompañar el workflow con documentación en `docs/infra-terraform.md` y con workflow template en `workflow-templates/`.

**Non-Goals:**

- Gestionar el estado remoto de Terraform (el backend es responsabilidad del consumidor).
- Soportar proveedores cloud distintos de AWS.
- Ejecutar `terraform destroy` (queda fuera del alcance de este workflow).
- Implementar drift detection automático.

## Decisions

### 1. Tres jobs con responsabilidad única: `validate` → `plan` → `apply`

**Decisión:** Separar en tres jobs independientes con dependencias explícitas (`needs`).

**Rationale:** Cada job tiene una responsabilidad clara y puede fallar de forma aislada. El validate no necesita backend real, lo que lo hace más rápido y seguro. El plan y el apply se ejecutan con backend real pero con credenciales OIDC de corta duración.

**Alternativa descartada:** Un único job con steps secuenciales — impide el gate de aprobación entre plan y apply.

### 2. Paso del tfplan entre jobs mediante `actions/upload-artifact` y `actions/download-artifact`

**Decisión:** El job `plan` sube el `tfplan` binario y el `job` apply lo descarga.

**Rationale:** Garantiza que el plan revisado y aprobado es exactamente el que se ejecuta en `apply`. Evita que el apply genere un plan nuevo con posibles diferencias.

**Alternativa descartada:** Regenerar el plan en `apply` — introduce una ventana de inconsistencia entre lo aprobado y lo ejecutado.

### 3. GitHub Environment como gate de aprobación para `apply`

**Decisión:** El job `apply` recibe el input `environment-name` y lo usa en `environment:`. Los revisores configurados en el environment de GitHub deben aprobar antes de que el job ejecute.

**Rationale:** Es el mecanismo nativo de GitHub Actions para aprobaciones manuales, con trazabilidad de quién aprobó y cuándo.

**Alternativa descartada:** Usar `workflow_dispatch` con confirmación manual — rompe el flujo automatizado de CI/CD.

### 4. Autenticación en AWS: OIDC o credenciales estáticas, con soporte opcional de role assumption

**Decisión:** El workflow soporta dos métodos de autenticación base via `aws-actions/configure-aws-credentials@v4`:
- **OIDC (recomendado):** sin credenciales estáticas. Requiere que el IAM Identity Provider OIDC esté configurado en la cuenta AWS.
- **Credenciales estáticas:** secrets `aws-access-key-id` y `aws-secret-access-key`.

En ambos casos, el input `aws-role-to-assume` es opcional e independiente: si se proporciona, se asume el role indicado tras la autenticación base, ya sea que esta haya sido por OIDC o por credenciales estáticas.

**Rationale:** OIDC es más seguro, pero algunos entornos heredados o cuentas AWS sin soporte OIDC configurado requieren credenciales estáticas. El role assumption es un paso adicional que aplica en ambos casos y permite el principio de mínimo privilegio.

**Riesgo:** Las credenciales estáticas tienen mayor superficie de ataque. Documentar claramente que OIDC es el método preferido.

### 5. Comentario automático del plan en Pull Requests

**Decisión:** En el job `plan`, cuando el evento es `pull_request`, se publica un comentario en el PR con el output de `terraform plan` usando `actions/github-script@v7`.

**Rationale:** Permite a los revisores ver el impacto exacto del cambio de infraestructura directamente en el PR sin necesidad de acceder a los logs del workflow.

**Alternativa descartada:** Usar una action de terceros específica para comentar planes de Terraform — agrega dependencia externa innecesaria cuando `github-script` es suficiente.

### 6. Input `backend-config` para configuración dinámica del backend

**Decisión:** El job `plan` y `apply` reciben el input `backend-config` como string con flags `-backend-config=key=value` que se pasan a `terraform init`.

**Rationale:** Permite que cada repositorio consumidor especifique su propio bucket S3, tabla DynamoDB y clave de estado sin modificar el workflow.

## Estructura de archivos

```
.github/workflows/infra-terraform.yml          # Reusable workflow
docs/infra-terraform.md                        # Documentación
workflow-templates/infra-terraform.yml         # Template para nuevos repos
workflow-templates/infra-terraform.properties.json  # Metadatos del template
```

### Inputs del workflow

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos .tf |
| `aws-region` | string | Sí | — | Región de AWS donde se despliega la infraestructura |
| `environment-name` | string | Sí | — | Nombre del GitHub Environment para el gate de aprobación |
| `backend-config` | string | No | `""` | Flags de configuración del backend (e.g. `-backend-config=bucket=my-bucket`) |
| `aws-role-to-assume` | string | No | `""` | ARN del IAM Role a asumir tras la autenticación base (compatible con OIDC y credenciales estáticas) |
| `enable-apply` | boolean | No | `true` | Habilita el job `apply`. Establecer en `false` para deshabilitar el apply (e.g. en Pull Requests) |

### Secrets del workflow

| Secret | Requerido | Descripción |
|---|---|---|
| `aws-access-key-id` | No* | AWS Access Key ID para autenticación con credenciales estáticas |
| `aws-secret-access-key` | No* | AWS Secret Access Key para autenticación con credenciales estáticas |

*Al menos uno de los métodos base debe estar configurado: OIDC (sin secrets de credenciales) o el par `aws-access-key-id` + `aws-secret-access-key`. El input `aws-role-to-assume` es opcional en ambos casos.

### Outputs del workflow

| Output | Descripción |
|---|---|
| `plan-exit-code` | Código de salida de terraform plan (0=sin cambios, 2=con cambios) |

## Risks / Trade-offs

- **Tamaño del artifact tfplan** → El tfplan binario puede ser grande en infraestructuras complejas. Mitigation: los artifacts de GitHub Actions soportan hasta 500 MB por defecto.
- **Expiración del artifact** → Por defecto los artifacts expiran en 90 días. Mitigation: documentar que el artifact del tfplan es temporal y sólo sirve para el apply inmediato.
- **OIDC no disponible en todos los repos** → El IAM Role debe tener configurado el trust policy correcto para el repositorio consumidor. Mitigation: documentar el trust policy requerido en `docs/infra-terraform.md`.
- **Backend no gestionado** → Si el consumidor no pasa `backend-config` correctamente, el init fallará. Mitigation: el input es opcional con string vacío por defecto y el validate usa `-backend=false`.
