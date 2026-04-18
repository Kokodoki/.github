## ADDED Requirements

### Requirement: Job build autentica en AWS y hace login en ECR

El workflow SHALL ejecutar un job `build` que autentique en AWS mediante OIDC o credenciales estáticas usando `aws-actions/configure-aws-credentials@v4` y luego realice login en Amazon ECR usando `aws-actions/amazon-ecr-login@v2`.

#### Scenario: Login exitoso con OIDC

- **WHEN** el OIDC Identity Provider está configurado en la cuenta AWS y el repositorio consumidor no provee secrets de credenciales estáticas
- **THEN** el step de configuración de credenciales AWS completa con éxito y el step de login a ECR obtiene un token de autenticación válido

#### Scenario: Login exitoso con credenciales estáticas

- **WHEN** los secrets `aws-access-key-id` y `aws-secret-access-key` están presentes y son válidos
- **THEN** la autenticación AWS con credenciales estáticas es exitosa y el login a ECR obtiene un token de autenticación válido

#### Scenario: Login exitoso con role assumption sobre OIDC

- **WHEN** el OIDC Identity Provider está configurado y el input `aws-role-to-assume` contiene un ARN de IAM Role válido
- **THEN** el workflow se autentica vía OIDC y asume el role indicado antes del login a ECR

#### Scenario: Login falla si ningún método de autenticación está configurado

- **WHEN** el input `aws-role-to-assume` está vacío y los secrets `aws-access-key-id` y `aws-secret-access-key` no están presentes
- **THEN** el step de configuración de credenciales AWS falla y el job termina con error sin intentar el login a ECR

---

### Requirement: Job build construye y publica la imagen Docker en ECR

El workflow SHALL construir la imagen Docker usando el `dockerfile` y `build-context` indicados, y publicarla en el repositorio ECR especificado con el tag `image-tag`, usando `docker/build-push-action@v6`.

#### Scenario: Build y push exitoso con inputs por defecto

- **WHEN** el repositorio consumidor provee solo `aws-region` y `ecr-repository`, y existe un `Dockerfile` en la raíz del repositorio
- **THEN** el workflow construye la imagen desde el `Dockerfile` en el contexto `.` y la publica en ECR con tag igual al SHA del commit

#### Scenario: Build y push con Dockerfile y contexto personalizados

- **WHEN** el repositorio consumidor provee `dockerfile: apps/api/Dockerfile` y `build-context: apps/api`
- **THEN** el workflow construye la imagen desde el Dockerfile y contexto indicados y la publica en ECR

#### Scenario: Build con build-args adicionales

- **WHEN** el repositorio consumidor provee `build-args` con uno o más pares `KEY=VALUE`
- **THEN** el workflow pasa los build args al proceso de build de Docker y la imagen se construye con esas variables de entorno disponibles en tiempo de build

#### Scenario: Build falla si el Dockerfile no existe en la ruta indicada

- **WHEN** el input `dockerfile` apunta a una ruta que no existe en el repositorio
- **THEN** el step de build falla con un error indicando que el Dockerfile no fue encontrado

#### Scenario: Build falla si el repositorio ECR no existe

- **WHEN** el input `ecr-repository` apunta a un repositorio que no existe en la cuenta AWS
- **THEN** el step de push falla con un error de ECR indicando que el repositorio no existe

---

### Requirement: Workflow expone outputs image-uri e image-digest

El workflow SHALL exponer como outputs el URI completo de la imagen publicada (`image-uri`) y el digest SHA256 de la imagen (`image-digest`), de manera que los workflows consumidores puedan encadenar pasos de despliegue.

#### Scenario: Outputs disponibles tras build exitoso

- **WHEN** el job `build` completa exitosamente y publica la imagen en ECR
- **THEN** el output `image-uri` contiene el URI completo en formato `<account>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>` y el output `image-digest` contiene el digest SHA256 de la imagen

#### Scenario: Workflow consumidor usa image-uri para despliegue

- **WHEN** un workflow consumidor invoca `build-docker` y luego referencia `needs.build-docker.outputs.image-uri`
- **THEN** obtiene el URI completo de la imagen publicada para usarlo en un step de despliegue posterior

---

### Requirement: Inputs, outputs y secrets en kebab-case

El workflow SHALL exponer todos sus inputs, outputs y secrets usando exclusivamente kebab-case.

#### Scenario: Consumidor pasa inputs en kebab-case

- **WHEN** un repositorio consumidor invoca el workflow usando `with: aws-region: "us-east-1"`, `with: ecr-repository: "mi-app"` y `with: image-tag: "v1.2.3"`
- **THEN** el workflow reconoce los inputs correctamente y los aplica en el job `build`

---

### Requirement: Documentación y workflow template acompañan al workflow

El workflow SHALL estar acompañado de un archivo de documentación en `docs/build-docker.md` y de un workflow template compuesto por `workflow-templates/build-docker.yml` y `workflow-templates/build-docker.properties.json`.

#### Scenario: Documentación describe todos los inputs, outputs, secrets y jobs

- **WHEN** un desarrollador consulta `docs/build-docker.md`
- **THEN** encuentra la descripción del propósito, un ejemplo de uso, la tabla de inputs/outputs/secrets y la descripción del job `build` y sus steps

#### Scenario: Workflow template referencia el reusable workflow con tag flotante

- **WHEN** un desarrollador usa el workflow template desde la UI de GitHub para crear un nuevo workflow en su repositorio
- **THEN** el archivo generado ya incluye `uses: <org>/.github/.github/workflows/build-docker.yml@v1` apuntando al último tag mayor
