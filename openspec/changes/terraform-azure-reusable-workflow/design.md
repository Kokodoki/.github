## Context

El repositorio `.github` centraliza reusable workflows de GitHub Actions consumidos por todos los repositorios de la organización. El workflow `infra-terraform.yml` ya implementa el patrón validate → plan → apply para AWS. Se requiere un workflow equivalente para Azure que siga la misma estructura y convenciones, autenticando mediante Workload Identity Federation (OIDC) o client secret de un Service Principal.

El nuevo workflow se llamará `infra-terraform-azure.yml` para cumplir la convención de prefijo `infra` y descriptor de tecnología/nube.

## Goals / Non-Goals

**Goals:**
- Reusable workflow con jobs `validate`, `plan` y `apply` para Terraform sobre Azure.
- Autenticación OIDC (Workload Identity Federation) como método principal; client secret como fallback.
- Misma estructura de artifacts (`tfplan`), gate de aprobación vía GitHub Environment y comentario en PRs que el workflow de AWS.
- Documentación en `docs/infra-terraform-azure.md` y workflow template en `workflow-templates/`.
- Todos los inputs/outputs/secrets en kebab-case.

**Non-Goals:**
- Soporte multi-cloud en un único workflow (AWS + Azure combinados).
- Modificación del workflow `infra-terraform.yml` existente (AWS).
- Gestión de la configuración del Entra ID App Registration o del backend de Terraform en Azure Storage.

## Decisions

### Decisión 1: Usar `azure/login@v2` para autenticación

**Elección:** `azure/login@v2` (action oficial de Microsoft).

**Rationale:** Es la action mantenida por Microsoft, compatible con OIDC (federated credentials) y con client secret. Produce las variables de entorno `ARM_*` que el provider de Terraform para Azure (`azurerm`) necesita de forma automática cuando se combina con `ARM_USE_OIDC=true` o `ARM_CLIENT_SECRET`.

**Alternativa descartada:** Configurar variables de entorno `ARM_*` manualmente — más verboso, más propenso a errores y no aprovecha la abstracción del action oficial.

---

### Decisión 2: OIDC como método primario, client secret como fallback

**Elección:** El workflow acepta los secrets `azure-client-id`, `azure-tenant-id`, `azure-subscription-id` para OIDC. Adicionalmente acepta `azure-client-secret` para autenticación con client secret. Si `azure-client-secret` está presente, se usa client secret; si no, se usa OIDC.

**Rationale:** OIDC elimina la rotación de credenciales de larga duración y es el estándar recomendado por Microsoft para CI/CD. El fallback a client secret permite onboarding progresivo sin bloquear equipos que aún no han configurado Workload Identity Federation.

**Alternativa descartada:** Solo OIDC — impediría adopción en organizaciones con restricciones en Entra ID.

---

### Decisión 3: Misma estructura de jobs que `infra-terraform.yml`

**Elección:** Jobs `validate`, `plan` y `apply` con las mismas responsabilidades que en el workflow de AWS.

**Rationale:** Consistencia entre workflows reduce la curva de aprendizaje. Los equipos que conocen el workflow de AWS pueden adoptar el de Azure sin fricción. El artefacto `tfplan` se sube y descarga de la misma manera.

---

### Decisión 4: Input `azure-location` no requerido en el workflow

**Elección:** No se incluye `azure-location` como input del workflow; es responsabilidad del código Terraform del repositorio consumidor.

**Rationale:** La ubicación (region) de los recursos es configuración de la infraestructura, no del pipeline. Forzarla como input del workflow acoplaría innecesariamente el workflow a la topología de la organización.

---

### Decisión 5: Backend de Terraform en Azure Blob Storage

**Elección:** El workflow acepta un input `backend-config` (string de flags) igual que el workflow de AWS, sin asumir el tipo de backend.

**Rationale:** Flexibilidad — algunos equipos pueden usar backends locales en validación o backends distintos a Azure Storage. La configuración específica del backend (`azurerm`, `storage_account_name`, etc.) se pasa mediante `backend-config`.

## Risks / Trade-offs

- **[Riesgo] Workload Identity Federation no configurado en Entra ID** → Los equipos deben registrar las credenciales federadas en su App Registration antes de usar OIDC. Mitigación: documentar los pasos de configuración en `docs/infra-terraform-azure.md`.
- **[Riesgo] `azure/login@v2` no establece `ARM_USE_OIDC` automáticamente en todas las versiones del provider** → Añadir `ARM_USE_OIDC: true` como variable de entorno en los jobs cuando se usa OIDC. Mitigación: incluir el env en el step de Configure Azure Credentials condicionalmente.
- **[Trade-off] Dos secrets separados para OIDC vs. uno para client secret** → Requiere que los consumidores pasen 3 secrets para OIDC (client-id, tenant-id, subscription-id). Aceptable porque es el estándar de `azure/login@v2`.

## Migration Plan

1. Crear `.github/workflows/infra-terraform-azure.yml`.
2. Crear `docs/infra-terraform-azure.md`.
3. Crear `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.
4. Los repositorios consumidores adoptan el nuevo workflow en sus propios workflows (`uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1`).
5. No hay rollback necesario — el nuevo workflow es independiente del de AWS.

## Open Questions

- ¿Debe el workflow soportar `azure-subscription-id` como input (no secret) para entornos donde el subscription ID no es sensible, o siempre como secret? Por defecto se modela como secret para mayor seguridad.
