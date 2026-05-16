## Context

El repositorio `.github` de la organización centraliza los reusable workflows de GitHub Actions. Ya existe `infra-terraform.yml` para AWS. Los equipos que trabajan con Azure no tienen un workflow estándar, lo que provoca duplicación de lógica en cada repositorio consumidor. Este diseño define cómo construir `infra-terraform-azure.yml` siguiendo los mismos patrones que el workflow de AWS.

## Goals / Non-Goals

**Goals:**
- Crear `infra-terraform-azure.yml` con jobs `validate`, `plan` y `apply` para infraestructura Terraform sobre Azure.
- Soportar autenticación OIDC (Federated Identity Credentials) y Service Principal con client secret mediante `azure/login@v2`.
- Configurar backend `azurerm` con Azure Storage Account a través de inputs.
- Publicar comentario de plan en Pull Requests.
- Requerir aprobación manual en el job `apply` mediante GitHub Environment.
- Proveer documentación en `docs/infra-terraform-azure.md` y workflow template en `workflow-templates/`.

**Non-Goals:**
- Soporte para otros providers de cloud en el mismo workflow.
- Gestión de Azure Storage Account o resource group para el backend (se asume que ya existen).
- Integración con Azure DevOps o pipelines distintos de GitHub Actions.
- Soporte para múltiples workspaces de Terraform en una misma ejecución.

## Decisions

### 1. Autenticación: `azure/login@v2` con OIDC o Service Principal

**Decisión**: Usar `azure/login@v2` como paso único de autenticación, que soporta tanto OIDC (via `client-id` sin `client-secret`) como Service Principal (via `client-id` + `client-secret`). Los tres secrets `azure-client-id`, `azure-tenant-id` y `azure-subscription-id` son siempre requeridos; `azure-client-secret` es opcional.

**Alternativas consideradas**:
- Usar `azure/login@v1`: descartado, versión desactualizada.
- Usar `az login` directamente en un step de shell: descartado, más frágil y sin soporte nativo de OIDC en GitHub Actions.

**Rationale**: `azure/login@v2` es el método oficial recomendado por Microsoft y GitHub para autenticación en workflows; abstrae OIDC y SP con la misma interfaz.

### 2. Backend: inputs explícitos para azurerm

**Decisión**: Exponer `backend-resource-group-name`, `backend-storage-account-name`, `backend-container-name` y `backend-key` como inputs individuales en lugar de un string libre de `-backend-config` flags (como hace el workflow de AWS).

**Alternativas consideradas**:
- Usar un único input `backend-config` como string libre: más flexible, pero propenso a errores y más difícil de documentar y validar.

**Rationale**: Los backends de `azurerm` tienen parámetros bien definidos y explícitos. Inputs individuales mejoran la legibilidad, la validación en tiempo de diseño y la experiencia del consumidor.

### 3. Estructura de jobs: validate → plan → apply (igual que infra-terraform.yml)

**Decisión**: Replicar la misma cadena de jobs del workflow de AWS, con `apply` condicionado por el input `enable-apply` y el GitHub Environment.

**Rationale**: Consistencia con el workflow existente facilita la adopción y el mantenimiento. Los equipos ya conocen el patrón.

### 4. Nombre del archivo: `infra-terraform-azure.yml`

**Decisión**: Prefijo `infra` (igual que el workflow de AWS) con sufijo `-azure` para distinguir el proveedor.

**Rationale**: Cumple las convenciones de nomenclatura del repositorio y es descriptivo sin ser redundante.

## Risks / Trade-offs

- **[Riesgo] El step `azure/login` falla si la Federated Identity Credential no está configurada en Azure AD** → Mitigación: documentar en `docs/infra-terraform-azure.md` los pasos para configurar la Federated Identity Credential en el App Registration de Azure.
- **[Trade-off] Inputs individuales de backend son menos flexibles que un string libre** → Los backends `azurerm` tienen parámetros estables; la flexibilidad sacrificada es mínima en la práctica.
- **[Riesgo] El artifact `tfplan` puede contener credenciales si el plan las imprime** → Mitigación: el artifact se retiene solo 7 días y el acceso está controlado por los permisos del repositorio.

## Migration Plan

1. Crear el archivo `.github/workflows/infra-terraform-azure.yml`.
2. Crear `docs/infra-terraform-azure.md`.
3. Crear `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.
4. No hay migración de workflows existentes; es una incorporación nueva.
5. Rollback: eliminar los tres archivos nuevos.

## Open Questions

- ¿Se debe usar `azure/login@v2` con `allow-no-subscriptions: true` para casos donde el scope es solo tenant? → Por ahora se asume que siempre hay una suscripción activa.
