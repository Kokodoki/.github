## ADDED Requirements

### Requirement: Workflow acepta disparadores via workflow_call
El workflow `build-docker.yml` SHALL definir `on: workflow_call` como único trigger, exponiendo inputs, outputs y secrets en kebab-case.

#### Scenario: Consumidor invoca el workflow con inputs mínimos
- **WHEN** un repositorio externo referencia el workflow con `uses` y provee al menos `image-name` y `image-tag`
- **THEN** el workflow se ejecuta correctamente con los valores por defecto para el resto de inputs

#### Scenario: Consumidor pasa inputs opcionales personalizados
- **WHEN** el consumidor provee `dockerfile`, `context`, `platforms` y `push`
- **THEN** el workflow utiliza esos valores en lugar de los defaults

### Requirement: Inputs del workflow en kebab-case
El workflow SHALL exponer los siguientes inputs tipados en kebab-case:
- `image-name` (string, requerido): nombre completo de la imagen (e.g. `myorg/myapp`)
- `image-tag` (string, requerido): tag de la imagen (e.g. `1.0.0`, `latest`)
- `registry-url` (string, opcional, default `ghcr.io`): URL del registry destino
- `context` (string, opcional, default `.`): directorio de contexto de build
- `dockerfile` (string, opcional, default `Dockerfile`): ruta al Dockerfile
- `platforms` (string, opcional, default `linux/amd64`): plataformas destino separadas por coma
- `push` (boolean, opcional, default `true`): si se publica la imagen al registry

#### Scenario: Input requerido no provisto
- **WHEN** el consumidor omite `image-name` o `image-tag`
- **THEN** GitHub Actions rechaza el dispatch y reporta error de validación de inputs

#### Scenario: Defaults aplicados para inputs opcionales
- **WHEN** el consumidor no provee `registry-url`, `context`, `dockerfile`, `platforms` ni `push`
- **THEN** el workflow usa `ghcr.io`, `.`, `Dockerfile`, `linux/amd64` y `true` respectivamente

### Requirement: Secrets del workflow en kebab-case
El workflow SHALL exponer los siguientes secrets:
- `registry-username` (requerido): usuario para autenticación en el registry
- `registry-password` (requerido): contraseña o token para autenticación

#### Scenario: Secrets provistos correctamente
- **WHEN** el consumidor pasa `registry-username` y `registry-password` válidos
- **THEN** el step de login al registry es exitoso y el build puede publicar la imagen

#### Scenario: Secrets no provistos
- **WHEN** el consumidor omite los secrets
- **THEN** el step de login falla con error de autenticación

### Requirement: Output image-digest
El workflow SHALL exponer el output `image-digest` con el digest SHA de la imagen construida.

#### Scenario: Build exitoso con push habilitado
- **WHEN** el build completa con `push: true`
- **THEN** el output `image-digest` contiene el digest SHA de la imagen publicada (e.g. `sha256:abc123...`)

#### Scenario: Build exitoso sin push
- **WHEN** el build completa con `push: false`
- **THEN** el output `image-digest` puede estar vacío o no disponible

### Requirement: Steps del job de build en orden correcto
El job SHALL ejecutar los steps en el siguiente orden: checkout del código, setup de QEMU, setup de Buildx, login al registry, build y push de la imagen.

#### Scenario: Ejecución completa con push habilitado
- **WHEN** se ejecuta el workflow con `push: true` y credenciales válidas
- **THEN** los steps se ejecutan secuencialmente y la imagen queda publicada en el registry

#### Scenario: Ejecución con push deshabilitado
- **WHEN** se ejecuta el workflow con `push: false`
- **THEN** los steps de checkout, QEMU, Buildx y build se ejecutan, pero no se publica la imagen ni se requiere autenticación exitosa

### Requirement: Documentación en /docs/build-docker.md
DEBE existir el archivo `/docs/build-docker.md` que documente: propósito, cómo consumirlo con ejemplo de uso, descripción de inputs/outputs/secrets y notas de versionado.

#### Scenario: Documentación contiene ejemplo de uso completo
- **WHEN** un desarrollador consulta `/docs/build-docker.md`
- **THEN** encuentra un bloque YAML con `uses`, `with` mostrando todos los inputs y `secrets` mostrando todos los secrets requeridos

### Requirement: Workflow template disponible
DEBEN existir los archivos `/workflow-templates/build-docker.yml` y `/workflow-templates/build-docker.properties.json` siguiendo las convenciones de GitHub.

#### Scenario: Template accesible desde la organización
- **WHEN** un repositorio de la organización crea un nuevo workflow desde la UI de GitHub
- **THEN** el template `build-docker` aparece como opción disponible

#### Scenario: Referencia en template apunta a tag estable
- **WHEN** se inspecciona el archivo `/workflow-templates/build-docker.yml`
- **THEN** la clave `uses` apunta a `<org>/.github/.github/workflows/build-docker.yml@<vX>` (tag semántico, nunca un branch)
