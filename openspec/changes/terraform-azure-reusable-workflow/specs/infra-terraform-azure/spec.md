## ADDED Requirements

### Requirement: Job validate ejecuta fmt, init sin backend y validate para Azure

El workflow SHALL ejecutar un job `validate` que corra secuencialmente `terraform fmt -check -recursive`, `terraform init -backend=false` y `terraform validate` sobre el directorio indicado por `working-directory`.

#### Scenario: Código Terraform con formato incorrecto falla en validate

- **WHEN** el repositorio consumidor invoca el workflow y algún archivo `.tf` no cumple el formato de `terraform fmt`
- **THEN** el job `validate` falla en el step de fmt y los jobs `plan` y `apply` no se ejecutan

#### Scenario: Configuración de Terraform inválida falla en validate

- **WHEN** los archivos `.tf` contienen errores de configuración o referencias inválidas al proveedor Azure
- **THEN** el job `validate` falla en el step de validate y los jobs `plan` y `apply` no se ejecutan

#### Scenario: Código Terraform válido supera validate

- **WHEN** los archivos `.tf` tienen formato correcto y configuración válida
- **THEN** el job `validate` termina con éxito y el job `plan` puede ejecutarse

---

### Requirement: Job plan genera tfplan sobre Azure Blob Storage y lo sube como artifact

El workflow SHALL ejecutar un job `plan` que autentique en Azure, inicialice el backend en Azure Blob Storage usando los inputs de backend, ejecute `terraform plan -out=tfplan` y suba el archivo `tfplan` como GitHub Actions Artifact con nombre `tfplan-<run_id>`.

#### Scenario: Plan con cambios genera artifact y exitcode 2

- **WHEN** el job `plan` se ejecuta y Terraform detecta diferencias entre el estado almacenado en Azure Blob Storage y la configuración deseada
- **THEN** se genera el archivo `tfplan`, se sube como artifact con nombre `tfplan-<run_id>` y el output `plan-exit-code` es `2`

#### Scenario: Plan sin cambios genera artifact con exitcode 0

- **WHEN** el job `plan` se ejecuta y no hay diferencias entre el estado y la configuración
- **THEN** se genera el archivo `tfplan`, se sube como artifact y el output `plan-exit-code` es `0`

#### Scenario: Plan falla si el backend de Azure Blob Storage no es accesible

- **WHEN** el Storage Account, Resource Group o container indicados en los inputs de backend no existen o la identidad no tiene permisos
- **THEN** el job `plan` falla en el step de init y el job `apply` no se ejecuta

---

### Requirement: Job apply descarga el tfplan y requiere aprobación via GitHub Environment

El workflow SHALL ejecutar un job `apply` que descargue el artifact `tfplan` generado por `plan`, requiera aprobación manual a través del GitHub Environment indicado en `environment-name` y ejecute `terraform apply tfplan`.

#### Scenario: Apply ejecuta exactamente el plan aprobado

- **WHEN** un revisor aprueba la ejecución en el GitHub Environment y el job `apply` arranca
- **THEN** el job descarga el artifact `tfplan` y ejecuta `terraform apply tfplan` sin generar un nuevo plan

#### Scenario: Apply bloqueado sin aprobación

- **WHEN** el job `apply` está pendiente y ningún revisor ha aprobado en el GitHub Environment
- **THEN** el job permanece en estado `waiting` y no ejecuta ningún comando de Terraform

#### Scenario: Apply se omite cuando enable-apply es false

- **WHEN** el input `enable-apply` es `false`
- **THEN** el job `apply` no se ejecuta, aunque `plan` haya completado exitosamente

#### Scenario: Apply falla si el artifact tfplan no existe

- **WHEN** el job `plan` no completó exitosamente y no subió el artifact
- **THEN** el job `apply` falla en el step de descarga del artifact

---

### Requirement: Autenticación en Azure mediante OIDC o Service Principal

El workflow SHALL soportar dos métodos de autenticación en Azure usando `azure/login@v2`:

- **OIDC (Workload Identity Federation)**: se proveen `azure-client-id`, `azure-tenant-id` y `azure-subscription-id`; el secret `azure-client-secret` no se pasa.
- **Service Principal con credenciales estáticas**: se proveen `azure-client-id`, `azure-tenant-id`, `azure-subscription-id` y el secret `azure-client-secret`.

#### Scenario: OIDC autentica correctamente sin client-secret

- **WHEN** el repositorio consumidor configura un Federated Identity Credential en Azure AD y provee `azure-client-id`, `azure-tenant-id` y `azure-subscription-id` sin `azure-client-secret`
- **THEN** la autenticación OIDC es exitosa y los comandos de Terraform pueden acceder a los recursos de Azure

#### Scenario: Service Principal con credenciales estáticas autentica correctamente

- **WHEN** los secrets `azure-client-id`, `azure-client-secret`, `azure-tenant-id` y `azure-subscription-id` están presentes y son válidos
- **THEN** la autenticación con Service Principal es exitosa y los comandos de Terraform pueden acceder a los recursos de Azure

#### Scenario: Autenticación falla con credenciales incorrectas

- **WHEN** las credenciales provistas son inválidas o el Service Principal no tiene los permisos necesarios en la suscripción
- **THEN** el step de `azure/login` falla y el job termina con error sin ejecutar Terraform

---

### Requirement: Job plan comenta el output del plan en Pull Requests

El workflow SHALL publicar un comentario en el Pull Request con el output de `terraform plan` cuando el evento disparador es `pull_request`, usando `permissions: pull-requests: write`. Si ya existe un comentario previo del bot para el mismo `working-directory`, SHALL actualizarlo en lugar de crear uno nuevo.

#### Scenario: PR recibe comentario con el plan al ejecutar el workflow

- **WHEN** el workflow es disparado por un evento `pull_request` y el job `plan` completa exitosamente
- **THEN** se publica o actualiza un comentario en el PR con el output completo de `terraform plan`

#### Scenario: No se comenta en eventos que no son PR

- **WHEN** el workflow es disparado por un evento distinto a `pull_request` (e.g. `push`, `workflow_dispatch`)
- **THEN** el step de comentario se omite sin error

---

### Requirement: Inputs, outputs y secrets en kebab-case

El workflow SHALL exponer todos sus inputs, outputs y secrets usando exclusivamente kebab-case.

#### Scenario: Consumidor pasa inputs en kebab-case

- **WHEN** un repositorio consumidor invoca el workflow usando inputs como `terraform-version`, `working-directory`, `environment-name` y `backend-storage-account-name`
- **THEN** el workflow reconoce los inputs correctamente y los aplica en cada job

---

### Requirement: Documentación y workflow template acompañan al workflow

El workflow SHALL estar acompañado de un archivo de documentación en `docs/infra-terraform-azure.md` y de un workflow template compuesto por `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

#### Scenario: Documentación describe todos los inputs, outputs, secrets y jobs

- **WHEN** un desarrollador consulta `docs/infra-terraform-azure.md`
- **THEN** encuentra la descripción del propósito, un ejemplo de uso, la tabla de inputs/outputs/secrets y la descripción de cada job y sus steps

#### Scenario: Workflow template referencia el reusable workflow con tag flotante

- **WHEN** un desarrollador usa el workflow template desde la UI de GitHub para crear un nuevo workflow en su repositorio
- **THEN** el archivo generado incluye `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1` apuntando al último tag mayor

#### Scenario: Workflow template usa $default-branch en los triggers

- **WHEN** el template es usado para crear un workflow en un repositorio con branch por defecto distinto a `main`
- **THEN** los triggers `on.push.branches` y `on.pull_request.branches` usan `$default-branch` para adaptarse al branch por defecto del repositorio
