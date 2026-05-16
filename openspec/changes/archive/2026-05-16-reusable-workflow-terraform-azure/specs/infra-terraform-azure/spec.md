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

El workflow SHALL ejecutar un job `plan` que corra `terraform init` con backend real usando `backend-config`, luego `terraform plan -out=tfplan` y suba el archivo `tfplan` binario junto con el output de texto como GitHub Actions Artifact.

#### Scenario: Plan con cambios genera artifact

- **WHEN** el job `plan` se ejecuta y Terraform detecta diferencias entre el estado actual y la configuración deseada
- **THEN** se genera el archivo `tfplan`, se sube como artifact con nombre `tfplan-<run_id>` y el output `plan-exit-code` es `2`

#### Scenario: Plan sin cambios genera artifact con código 0

- **WHEN** el job `plan` se ejecuta y no hay diferencias entre el estado y la configuración
- **THEN** se genera el archivo `tfplan`, se sube como artifact y el output `plan-exit-code` es `0`

#### Scenario: Plan falla si el backend no es accesible

- **WHEN** la configuración de `backend-config` apunta a un Storage Account de Azure inexistente o sin permisos
- **THEN** el job `plan` falla en el step de init y el job `apply` no se ejecuta

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

### Requirement: Job apply condicionado por el input enable-apply

El workflow SHALL ejecutar el job `apply` únicamente cuando el input booleano `enable-apply` sea `true`. Si `enable-apply` es `false`, el job `apply` se omite por completo.

#### Scenario: Apply se omite cuando enable-apply es false

- **WHEN** el repositorio consumidor invoca el workflow con `enable-apply: false`
- **THEN** el job `apply` no se ejecuta y el workflow termina exitosamente tras el job `plan`

#### Scenario: Apply se ejecuta cuando enable-apply es true

- **WHEN** el repositorio consumidor invoca el workflow con `enable-apply: true` (o sin especificarlo, usando el valor por defecto)
- **THEN** el job `apply` se ejecuta tras el job `plan` con la protección del GitHub Environment

---

### Requirement: Autenticación en Azure mediante OIDC con azure/login

El workflow SHALL autenticar en Azure usando exclusivamente OIDC (Workload Identity Federation) mediante `azure/login@v2`, usando los inputs `azure-client-id`, `azure-tenant-id` y `azure-subscription-id`. No se soportan credenciales estáticas (client secret).

#### Scenario: OIDC autentica correctamente con Federated Credential configurado

- **WHEN** el repositorio consumidor provee `azure-client-id`, `azure-tenant-id` y `azure-subscription-id` válidos y el Federated Credential está configurado en Entra ID para ese repositorio
- **THEN** la autenticación OIDC es exitosa y los comandos de Terraform pueden acceder a los recursos de Azure

#### Scenario: Autenticación falla si el Federated Credential no está configurado

- **WHEN** el Federated Credential no existe en la aplicación de Entra ID para el repositorio y rama que ejecutan el workflow
- **THEN** el step de `azure/login` falla con error de autenticación y el job termina sin ejecutar Terraform

#### Scenario: Autenticación en job plan y apply se realiza de forma independiente

- **WHEN** el job `plan` y el job `apply` se ejecutan en distintos runners
- **THEN** cada job realiza su propio step de `azure/login` usando los mismos inputs antes de ejecutar comandos de Terraform

---

### Requirement: Inputs, outputs y secrets en kebab-case

El workflow SHALL exponer todos sus inputs, outputs y secrets usando exclusivamente kebab-case.

#### Scenario: Consumidor pasa inputs en kebab-case

- **WHEN** un repositorio consumidor invoca el workflow usando `with: terraform-version: "1.9.0"` y `with: azure-client-id: "..."`
- **THEN** el workflow reconoce los inputs correctamente y los aplica en cada job

---

### Requirement: Documentación y workflow template acompañan al workflow

El workflow SHALL estar acompañado de un archivo de documentación en `docs/infra-terraform-azure.md` y de un workflow template compuesto por `workflow-templates/infra-terraform-azure.yml` y `workflow-templates/infra-terraform-azure.properties.json`.

#### Scenario: Documentación describe todos los inputs, outputs, secrets y jobs

- **WHEN** un desarrollador consulta `docs/infra-terraform-azure.md`
- **THEN** encuentra la descripción del propósito, un ejemplo de uso, la tabla de inputs/outputs/secrets y la descripción de cada job y sus steps

#### Scenario: Workflow template referencia el reusable workflow con tag flotante

- **WHEN** un desarrollador usa el workflow template desde la UI de GitHub para crear un nuevo workflow en su repositorio
- **THEN** el archivo generado ya incluye `uses: <org>/.github/.github/workflows/infra-terraform-azure.yml@vX` apuntando al último tag mayor
