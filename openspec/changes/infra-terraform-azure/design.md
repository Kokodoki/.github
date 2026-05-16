## Context

El repositorio `.github` centraliza reusable workflows para la organización. Ya existe `infra-terraform.yml` para AWS. Los equipos que despliegan infraestructura en **Azure** necesitan un workflow equivalente que reutilice los mismos patrones (validate → plan → apply) pero con autenticación OIDC contra Azure Active Directory en lugar de AWS IAM.

El workflow debe seguir las convenciones ya establecidas: kebab-case, prefijo `infra`, documentación en `/docs/`, workflow template en `/workflow-templates/`.

## Goals / Non-Goals

**Goals:**
- Autenticación en Azure mediante OIDC (Workload Identity Federation) con `azure/login`.
- Job `validate` independiente de credenciales (solo formato y sintaxis de Terraform).
- Job `plan` con backend de Azure (Azure Blob Storage), artifact del plan y comentario en PRs.
- Job `apply` condicional via input `enable-apply`, protegido por GitHub Environment.
- Todos los inputs/secrets en kebab-case.
- Documentación completa y workflow template.

**Non-Goals:**
- Soporte para otros proveedores cloud (AWS, GCP).
- Gestión del backend storage (el consumidor provee la config via `backend-config`).
- Creación automática del GitHub Environment o del App Registration en Azure.

## Decisions

### Autenticación: OIDC con `azure/login`

**Decisión:** Usar `azure/login@v2` con los inputs `client-id`, `tenant-id` y `subscription-id`.  
**Alternativas consideradas:**
- Credenciales estáticas (`client-secret`): Se ofrece como secret **opcional** `azure-client-secret`. Si se provee, se pasa al action; si no, se usa OIDC puro.
- Service Principal con certificado: Complejidad innecesaria para el caso de uso general.  
**Rationale:** OIDC elimina la gestión de secretos de larga duración. El input `azure-client-secret` se acepta opcionalmente para equipos que no han configurado Workload Identity Federation.

### Estructura de jobs: misma que `infra-terraform`

**Decisión:** `validate` → `plan` → `apply` con `needs` encadenados.  
**Rationale:** Coherencia con el workflow AWS existente. Los equipos ya conocen este patrón.

### Comentario en PRs: `actions/github-script`

**Decisión:** Usar `actions/github-script@v7` para crear o actualizar el comentario del plan.  
**Rationale:** Sin dependencias externas adicionales; el token de GitHub ya está disponible.

### Condición del job `apply`

**Decisión:** `if: inputs.enable-apply` booleano.  
**Rationale:** Permite a los consumidores deshabilitar el apply en contextos de PR donde solo se quiere validar y planificar.

### Versiones de actions

Se usan los tags flotantes mayores más recientes:
- `actions/checkout@v4`
- `hashicorp/setup-terraform@v3`
- `azure/login@v2`
- `actions/upload-artifact@v4`
- `actions/download-artifact@v4`
- `actions/github-script@v7`

## Risks / Trade-offs

- **OIDC requiere configuración previa en Azure** → El consumidor debe crear el App Registration y configurar Federated Credentials. La documentación debe incluir instrucciones claras.
- **`azure-client-secret` opcional aumenta la superficie de secretos** → Se documenta explícitamente como opción secundaria y no recomendada. La recomendación es OIDC.
- **Tamaño del output del plan** → Se trunca a 65 000 caracteres en el comentario del PR (mismo límite que el workflow AWS) para no exceder el límite de la API de GitHub.
