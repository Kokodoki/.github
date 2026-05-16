## ADDED Requirements

### Requirement: Job validate ejecuta fmt, init sin backend y validate

El workflow SHALL ejecutar un job `validate` que corra secuencialmente `terraform fmt -check -recursive`, `terraform init -backend=false` y `terraform validate` sobre el directorio indicado por `working-directory`.

#### Scenario: Código Terraform con formato incorrecto falla en validate

- **WHEN** el repositorio consumidor invoca el workflow y algún archivo `.tf` no cumple el formato de `terraform fmt`
- **THEN** el job `validate` falla en el step de fmt y los jobs `plan` y `apply` no se ejecutan

#### Scenario: Configuración de Terraform inválida falla en validate

- **WHEN** el repositorio consumidor invoca el workflow y los archivos `.tf` contienen errores de configuración
- **THEN** el job `validate` falla en el step de validate y los jobs `plan` y `apply` no se ejecutan

#### Scenario: Código Terraform válido supera validate

- **WHEN** los archivos `.tf` tienen formato correcto y configuración válida
- **THEN** el job `validate` termina con éxito y el job `plan` puede ejecutarse

---

### Requirement: Job plan genera tfplan y lo sube como artifact

El workflow SHALL ejecutar un job `plan` que corra `terraform init` con backend `azurerm` usando los inputs de backend, luego `terraform plan -out=tfplan` y suba el archivo `tfplan` binario como GitHub Actions Artifact.

#### Scenario: Plan con cambios genera artifact

- **WHEN** el job `plan` se ejecuta y Terraform detecta diferencias entre el estado actual y la configuración deseada
- **THEN** se genera el archivo `tfplan`, se sube como artifact con nombre `tfplan-<run_id>` y el output `plan-exit-code` es `2`

#### Scenario: Plan sin cambios genera artifact con código 0

- **WHEN** el job `plan` se ejecuta y no hay diferencias entre el estado y la configuración
- **THEN** se genera el archivo `tfplan`, se sube como artifact y el output `plan-exit-code` es `0`

#### Scenario: Plan falla si el backend de Azure Storage no es accesible

- **WHEN** los inputs de backend apuntan a un Storage Account o container inexistente en Azure
- **THEN** el job `plan` falla en el step de init y el job `apply` no se ejecuta

---

### Requirement: Job apply descarga el tfplan y requiere aprobación via GitHub Environment

El workflow SHALL ejecutar un job `apply` que descargue el artifact `tfplan` generado por `plan`, requiera aprobación manual a través del GitHub Environment indicado en `environment-name` y ejecute `terraform apply tfplan`.

#### Scenario: Apply ejecuta exactamente el plan aprobado

- **WHEN** un revisor aprueba la ejecución en el GitHub Environment y el job `apply` arranca
- **THEN** el job descarga el artifact `tfplan` del job `plan` y ejecuta `terraform apply tfplan` sin generar un plan nuevo

#### Scenario: Apply bloqueado sin aprobación

- **WHEN** el job `apply` está pendiente y ningún revisor ha aprobado en el GitHub Environment
- **THEN** el job permanece en estado `waiting` y no ejecuta ningún comando de Terraform

#### Scenario: Apply falla si el artifact tfplan no existe

- **WHEN** el job `plan` no completó exitosamente y no subió el artifact
- **THEN** el job `apply` falla en el step de descarga del artifact

---

### Requirement: Autenticación en Azure mediante OIDC o Service Principal

El workflow SHALL soportar dos métodos de autenticación en Azure usando `azure/login@v2`: OIDC (Federated Identity Credentials, sin `azure-client-secret`) o Service Principal con client secret (con `azure-client-secret`). Los secrets `azure-client-id`, `azure-tenant-id` y `azure-subscription-id` son siempre requeridos.

#### Scenario: OIDC autentica correctamente sin client secret

- **WHEN** el repositorio consumidor no provee el secret `azure-client-secret` y la Federated Identity Credential está configurada en el App Registration de Azure AD
- **THEN** la autenticación OIDC es exitosa y los comandos de Terraform pueden acceder a los recursos de Azure

#### Scenario: Service Principal con client secret autentica correctamente

- **WHEN** los secrets `azure-client-id`, `azure-tenant-id`, `azure-subscription-id` y `azure-client-secret` están presentes y son válidos
- **THEN** la autenticación con Service Principal es exitosa y los comandos de Terraform pueden acceder a los recursos de Azure

#### Scenario: Autenticación falla si los secrets requeridos no están presentes

- **WHEN** alguno de los secrets `azure-client-id`, `azure-tenant-id` o `azure-subscription-id` no está presente
- **THEN** el step de autenticación Azure falla y el job termina con error

---

### Requirement: Backend azurerm configurado mediante inputs individuales

El workflow SHALL configurar el backend `azurerm` de Terraform usando los inputs `backend-resource-group-name`, `backend-storage-account-name`, `backend-container-name` y `backend-key` pasados como flags `-backend-config` al comando `terraform init`.

#### Scenario: Backend se inicializa correctamente con los inputs provistos

- **WHEN** el repositorio consumidor provee los cuatro inputs de backend con valores válidos
- **THEN** el step `terraform init` configura el backend `azurerm` y el estado remoto se almacena en el Storage Account indicado

#### Scenario: Init falla si el Storage Account no existe

- **WHEN** el input `backend-storage-account-name` apunta a un Storage Account inexistente en la suscripción de Azure
- **THEN** el step de `terraform init` falla con error de backend y el job termina

---

### Requirement: Job plan comenta el output del plan en Pull Requests

El workflow SHALL publicar un comentario en el Pull Request con el output de `terraform plan` cuando el evento disparador es `pull_request`, usando `permissions: pull-requests: write`.

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

- **WHEN** un repositorio consumidor invoca el workflow usando `with: terraform-version: "1.9.0"` y `with: backend-storage-account-name: "mystorageaccount"`
- **THEN** el workflow reconoce los inputs correctamente y los aplica en cada job

---

### Requirement: Documentación y workflow template acompañan al workflow

El workflow SHALL estar acompañado de un archivo de documentación en `docs/infra-terraform-azure.md` y de un workflow template compuesto por `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

#### Scenario: Documentación describe todos los inputs, outputs, secrets y jobs

- **WHEN** un desarrollador consulta `docs/infra-terraform-azure.md`
- **THEN** encuentra la descripción del propósito, un ejemplo de uso, la tabla de inputs/outputs/secrets y la descripción de cada job y sus steps

#### Scenario: Workflow template referencia el reusable workflow con tag flotante

- **WHEN** un desarrollador usa el workflow template desde la UI de GitHub para crear un nuevo workflow en su repositorio
- **THEN** el archivo generado ya incluye `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@v1` apuntando al último tag mayor
