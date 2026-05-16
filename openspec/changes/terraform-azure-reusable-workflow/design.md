## Context

El repositorio ya provee `infra-terraform.yml` para AWS (validate → plan → apply con OIDC/credenciales estáticas, backend S3, comentario en PR y gate de aprobación). Los equipos que usan Azure con Terraform mantienen sus propios workflows ad-hoc sin estándares organizacionales. El nuevo workflow `infra-terraform-azure.yml` replica la misma estructura pero adaptada al proveedor Azure: autenticación vía `azure/login`, backend sobre Azure Blob Storage y los inputs/secrets propios del SDK de Azure.

## Goals / Non-Goals

**Goals:**
- Proveer un reusable workflow con los jobs `validate`, `plan` y `apply` para Terraform en Azure.
- Soportar autenticación mediante OIDC (Workload Identity Federation) y Service Principal con credenciales estáticas.
- Usar Azure Blob Storage como backend de estado de Terraform.
- Publicar el output del plan como comentario en Pull Requests.
- Requerir aprobación manual vía GitHub Environment antes del apply.
- Acompañar con documentación en `docs/` y workflow template en `workflow-templates/`.

**Non-Goals:**
- Soporte multi-cloud en un único workflow (cada proveedor tiene su propio archivo).
- Gestión de recursos de Azure (Resource Groups, Storage Accounts) requeridos por el backend — son responsabilidad del consumidor.
- Soporte para autenticación con Azure Managed Identity desde runners auto-hospedados (fuera de alcance inicial).

## Decisions

### 1. Archivo separado para Azure (`infra-terraform-azure.yml`) en lugar de parametrizar el workflow existente

**Decisión**: Crear un nuevo archivo independiente, no modificar `infra-terraform.yml`.

**Rationale**: Los inputs, secrets y actions de AWS y Azure son incompatibles; unificarlos en un solo workflow requeriría condiciones complejas y haría el contrato del workflow difícil de entender para los consumidores. La paridad de estructura (mismos jobs, misma lógica de flujo) ya reduce la duplicación conceptual.

**Alternativa considerada**: Parametrizar el workflow existente con un input `cloud-provider`. Descartada por complejidad de la matriz de inputs/secrets y por riesgo de romper consumidores existentes de `infra-terraform.yml`.

---

### 2. Autenticación: `azure/login@v2` con soporte OIDC y Service Principal

**Decisión**: Usar `azure/login@v2` como único step de autenticación. Soporta ambos métodos con la misma firma de inputs.

- **OIDC**: el consumidor configura un Federated Identity Credential en Azure AD; los secrets `azure-client-secret` no se pasan.
- **Service Principal**: se proveen `azure-client-id`, `azure-client-secret`, `azure-tenant-id` y `azure-subscription-id` como secrets.

**Rationale**: La action oficial de Azure Login unifica ambos flujos con un solo step. OIDC es el método recomendado por su menor superficie de ataque (sin secretos de larga duración).

---

### 3. Backend: Azure Blob Storage vía inputs `backend-resource-group-name`, `backend-storage-account-name`, `backend-container-name`, `backend-key`

**Decisión**: Exponer los cuatro parámetros del backend de Azure como inputs opcionales; si no se proveen, se permite configuración en los archivos `.tf` del consumidor mediante `backend-config` genérico (igual que en el workflow AWS).

**Rationale**: El backend de Azure requiere más parámetros que el de AWS. Exponerlos como inputs primero-clase mejora la experiencia del consumidor y permite validación temprana.

---

### 4. Estructura de jobs idéntica a `infra-terraform.yml`

**Decisión**: `validate → plan → apply`, con el mismo patrón de upload/download de artifacts, comentario en PR y gate de aprobación.

**Rationale**: Consistencia organizacional. Los equipos que ya conocen el workflow AWS adoptan el de Azure sin fricción.

---

### 5. Nombre del archivo: `infra-terraform-azure.yml`

**Decisión**: Prefijo `infra` (infraestructura), descriptor `terraform-azure` para distinguirlo del workflow AWS.

**Rationale**: Sigue la convención de nomenclatura del repositorio (`infra-terraform.yml` para AWS → `infra-terraform-azure.yml` para Azure).

## Risks / Trade-offs

- **Permisos de OIDC en GitHub Actions** → El consumidor debe agregar `permissions: id-token: write` en su workflow caller. Mitigación: documentarlo en `docs/` y en el template.
- **Recursos de Azure para el backend deben existir previamente** → Si el Storage Account o container no existen, `terraform init` falla. Mitigación: indicarlo como prerrequisito en la documentación.
- **Duplicación de lógica de comentario en PR** → El script de `actions/github-script` es casi idéntico al de `infra-terraform.yml`. Mitigación: aceptada como trade-off de mantener workflows independientes; en el futuro se podría extraer a una composite action.
- **Versiones de actions de terceros** → `azure/login`, `hashicorp/setup-terraform` pueden tener breaking changes. Mitigación: usar tags flotantes mayores (`@v2`, `@v4`) y revisar en cada actualización.

## Migration Plan

1. Crear `.github/workflows/infra-terraform-azure.yml` con los jobs `validate`, `plan`, `apply`.
2. Crear `docs/infra-terraform-azure.md` con descripción, ejemplo de uso y tabla de inputs/outputs/secrets.
3. Crear `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.
4. Los repositorios consumidores adoptan el workflow referenciando `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1`.
5. No hay migración de workflows existentes; este workflow es nuevo.

## Open Questions

- ¿Se requiere soporte para múltiples suscripciones de Azure en un solo run? (Fuera de alcance inicial, pero podría necesitar el input `azure-subscription-id` como requerido vs. opcional.)
- ¿El tag flotante inicial debe ser `@v1` o seguir el esquema de versiones ya existente en el repositorio?
