## Context

El repositorio `.github` de la organización ya dispone de `infra-terraform.yml`, un reusable workflow de Terraform para AWS. La organización requiere ahora un workflow equivalente para Azure. A diferencia del workflow AWS, la autenticación en Azure se realiza **exclusivamente mediante OIDC** (Workload Identity Federation), eliminando la necesidad de gestionar credenciales estáticas. El workflow sigue el mismo patrón de tres etapas que el existente: `validate → plan → apply`.

## Goals / Non-Goals

**Goals:**
- Autenticar en Azure de forma segura y sin secretos estáticos usando OIDC (`azure/login@v2`).
- Ejecutar un job `validate` (fmt, init sin backend, validate) que bloquee el pipeline ante código incorrecto.
- Ejecutar un job `plan` que genere el artifact `tfplan`, calcule el exit code y publique el output en comentarios de Pull Requests.
- Ejecutar un job `apply` condicionado por el flag `enable-apply` y protegido por un GitHub Environment para aprobación manual.
- Proveer documentación en `docs/infra-terraform-azure.md` y workflow template en `workflow-templates/`.
- Mantener la misma convención de nomenclatura (kebab-case) y estructura que `infra-terraform.yml`.

**Non-Goals:**
- Soporte para credenciales estáticas de Azure (Service Principal con client secret). Solo OIDC.
- Reemplazar o modificar el workflow existente de AWS (`infra-terraform.yml`).
- Soporte para backends de Terraform distintos al Azure Backend (azurerm).
- Soporte multi-cloud en un solo workflow.

## Decisions

### D1 — Autenticación exclusiva con OIDC (azure/login@v2)

**Decisión**: Se usa únicamente OIDC (Workload Identity Federation) para autenticación en Azure. Los tres inputs requeridos son `azure-client-id`, `azure-tenant-id` y `azure-subscription-id`.

**Alternativas consideradas**:
- _Service Principal con client secret_: descartado porque requiere gestionar un secreto de larga duración en GitHub, aumentando el riesgo de exposición.
- _Credenciales mixtas (OIDC + fallback estático) como en AWS_: descartado para simplificar el workflow y forzar la adopción de OIDC desde el inicio.

**Rationale**: OIDC elimina secretos estáticos, reduce la superficie de ataque y es la recomendación oficial de Microsoft para GitHub Actions con Azure.

### D2 — Inputs `azure-client-id`, `azure-tenant-id`, `azure-subscription-id` como inputs (no secrets)

**Decisión**: Los tres identificadores de la aplicación registrada en Entra ID se pasan como `inputs` de tipo `string`, no como `secrets`.

**Alternativas consideradas**:
- _Pasar como secrets_: descartado porque estos valores no son secretos sensibles (son IDs públicos de la aplicación) y pasar como inputs facilita su visibilidad en los logs de debug y su uso en condiciones.

**Rationale**: Sigue la documentación oficial de `azure/login@v2` y la práctica recomendada por Microsoft.

### D3 — Estructura de jobs idéntica a `infra-terraform.yml`

**Decisión**: Los tres jobs (`validate`, `plan`, `apply`) siguen el mismo orden, nombres y responsabilidades que el workflow AWS. Solo cambia el step de autenticación.

**Rationale**: Consistencia interna para los equipos que ya conocen el workflow AWS. Facilita el mantenimiento y la evolución paralela de ambos workflows.

### D4 — Comentario del plan en PRs usando `actions/github-script`

**Decisión**: Se usa el mismo mecanismo que en `infra-terraform.yml`: `actions/github-script@v7` para crear o actualizar un comentario con el output del plan. El comentario se identifica por el `working-directory` para evitar duplicados.

### D5 — Nombre del archivo: `infra-terraform-azure.yml`

**Decisión**: Se sigue el prefijo `infra` (prefijo permitido) con el descriptor `terraform-azure`, resultando en `infra-terraform-azure.yml`.

**Rationale**: Coherencia con `infra-terraform.yml` y las convenciones de nomenclatura del repositorio.

## Risks / Trade-offs

- **[Riesgo] Configuración incorrecta del Federated Credential en Entra ID** → La autenticación OIDC fallará si el Federated Credential no está configurado para el repositorio y la rama/ambiente correctos. _Mitigación_: La documentación incluirá los pasos necesarios para configurar el Federated Credential en Azure.
- **[Trade-off] No hay fallback a credenciales estáticas** → Equipos que aún no hayan configurado OIDC no podrán usar este workflow. _Mitigación_: La documentación detalla los prerequisitos claramente.
- **[Riesgo] Cambios en la API de `azure/login`** → Versiones futuras de `azure/login` podrían cambiar los inputs requeridos. _Mitigación_: Se usa el tag flotante mayor (`@v2`) para recibir patches automáticamente.

## Migration Plan

No hay migración requerida. Este es un nuevo archivo que no modifica ningún workflow existente. Los equipos adoptarán el nuevo workflow de forma voluntaria siguiendo la documentación.

## Open Questions

_Ninguna pendiente al momento de este diseño._
