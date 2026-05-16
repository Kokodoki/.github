## ADDED Requirements

### Requirement: Reusable workflow de Maven publicable por workflow_call
El sistema SHALL proveer un reusable workflow en `.github/workflows/build-maven-jfrog.yml` invocable mediante `on: workflow_call` para proyectos Java con Maven.

#### Scenario: Repositorio consumidor invoca el workflow correctamente
- **WHEN** un repositorio define un job con `uses: <org>/.github/.github/workflows/build-maven-jfrog.yml@<tag-mayor>` y pasa los parámetros requeridos
- **THEN** el workflow se ejecuta sin requerir lógica duplicada en el repositorio consumidor

### Requirement: Inputs, outputs y secrets en kebab-case con contrato explícito
El workflow SHALL definir inputs, outputs y secrets en kebab-case, incluyendo tipo, obligatoriedad y descripción para build y publicación en JFrog.

#### Scenario: Parámetros de entrada válidos son aceptados
- **WHEN** el consumidor envía inputs en kebab-case y provee los secrets requeridos
- **THEN** el workflow reconoce los parámetros y continúa el pipeline sin errores de validación de contrato

### Requirement: Job de build compila con Maven
El workflow SHALL ejecutar un job de build que configure Java, prepare Maven y ejecute la compilación del proyecto con argumentos configurables.

#### Scenario: Build exitoso genera artefacto Maven
- **WHEN** el proyecto contiene configuración Maven válida y dependencias accesibles
- **THEN** el job de build finaliza exitosamente y deja listo el artefacto para publicación

#### Scenario: Build falla ante errores de compilación
- **WHEN** el código fuente o la configuración Maven contiene errores
- **THEN** el job de build falla y el job de publicación no se ejecuta

### Requirement: Job de publicación sube artefacto a JFrog Artifactory
El workflow SHALL ejecutar un job de publicación que autentique contra JFrog y publique el artefacto Maven en el repositorio objetivo configurado.

#### Scenario: Publicación exitosa en JFrog
- **WHEN** las credenciales y el endpoint de JFrog son válidos y el build fue exitoso
- **THEN** el artefacto se publica en el repositorio configurado de JFrog y el workflow finaliza con éxito

#### Scenario: Publicación falla por credenciales inválidas
- **WHEN** los secrets de autenticación JFrog son incorrectos o carecen de permisos
- **THEN** el job de publicación falla y reporta error de autenticación/autorización

### Requirement: Workflow expone outputs de trazabilidad de publicación
El workflow SHALL exponer outputs en kebab-case con metadatos de la publicación del artefacto para su consumo en workflows downstream.

#### Scenario: Outputs disponibles tras publicación exitosa
- **WHEN** el job de publicación termina correctamente
- **THEN** el workflow expone outputs con la coordenada del artefacto y la URL de referencia en JFrog

### Requirement: El workflow incluye documentación y workflow template
El cambio SHALL incluir documentación en `docs/build-maven-jfrog.md` y workflow template en `workflow-templates/build-maven-jfrog.yml` y `workflow-templates/build-maven-jfrog.properties.json`.

#### Scenario: Documentación cubre interfaz y uso
- **WHEN** un desarrollador consulta la documentación del workflow
- **THEN** encuentra propósito, ejemplo de consumo, tabla de inputs/outputs/secrets y descripción de jobs/steps

#### Scenario: Template genera consumo correcto del reusable
- **WHEN** un desarrollador crea un workflow desde el template
- **THEN** el archivo generado referencia el reusable workflow mediante tag mayor y usa `$default-branch` únicamente en triggers permitidos
