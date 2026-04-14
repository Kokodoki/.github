## ADDED Requirements

### Requirement: Workflow acepta llamadas de otros repositorios vía workflow_call
El workflow `scan-sonar.yml` SHALL exponerse con `on: workflow_call` para ser consumido por cualquier repositorio de la organización.

#### Scenario: Consumidor llama al workflow con inputs requeridos
- **WHEN** un repositorio llama al reusable workflow proporcionando `sonar-project-key` y `sonar-organization`
- **THEN** el workflow se ejecuta correctamente sin errores de configuración

---

### Requirement: Inputs del workflow
El workflow SHALL aceptar los siguientes inputs:
- `sonar-project-key` (string, requerido): clave única del proyecto en SonarCloud.
- `sonar-organization` (string, requerido): organización en SonarCloud.
- `main-branch` (string, opcional, default `main`): rama principal del repositorio.
- `extra-args` (string, opcional, default vacío): argumentos adicionales para el scanner de SonarCloud.

#### Scenario: Input requerido faltante
- **WHEN** el consumidor llama al workflow sin proporcionar `sonar-project-key` o `sonar-organization`
- **THEN** GitHub Actions rechaza la llamada con un error de validación de inputs antes de ejecutar el job

#### Scenario: Uso con solo inputs requeridos
- **WHEN** el consumidor proporciona únicamente `sonar-project-key` y `sonar-organization`
- **THEN** el workflow usa `main` como `main-branch` y no pasa argumentos extra al scanner

---

### Requirement: Secret SONAR_TOKEN requerido
El workflow SHALL requerir el secret `sonar-token` (mapeado desde `SONAR_TOKEN` del repositorio o de la organización) para autenticarse con SonarCloud.

#### Scenario: Secret presente y válido
- **WHEN** el consumidor proporciona el secret `sonar-token` con un token válido de SonarCloud
- **THEN** el scanner se autentica correctamente y ejecuta el análisis

#### Scenario: Secret ausente
- **WHEN** el consumidor no proporciona el secret `sonar-token`
- **THEN** el job falla con un error de autenticación de SonarCloud

---

### Requirement: Ejecución del análisis de SonarCloud
El workflow SHALL ejecutar el análisis de código usando la action oficial `SonarSource/sonarcloud-github-action@v3` con `sonar.qualitygate.wait=true`.

#### Scenario: Análisis exitoso
- **WHEN** el scanner completa el análisis sin errores
- **THEN** el job continúa a la evaluación del Quality Gate

#### Scenario: Error durante el análisis
- **WHEN** el scanner encuentra un error fatal (ej. configuración inválida, proyecto no encontrado)
- **THEN** el job falla con el error del scanner

---

### Requirement: Validación del Quality Gate
El workflow SHALL fallar el job si el Quality Gate de SonarCloud no pasa, usando `sonar.qualitygate.wait=true`.

#### Scenario: Quality Gate aprobado
- **WHEN** SonarCloud evalúa el análisis y el Quality Gate tiene estado `OK`
- **THEN** el job finaliza con éxito (exit code 0)

#### Scenario: Quality Gate rechazado
- **WHEN** SonarCloud evalúa el análisis y el Quality Gate tiene estado `ERROR`
- **THEN** el job finaliza con fallo (exit code distinto de 0)

---

### Requirement: Documentación del workflow
El workflow SHALL acompañarse de un archivo de documentación en `/docs/scan-sonar.md` que incluya: propósito, cómo consumirlo (ejemplo de uso), descripción de cada job y sus steps, e inputs/outputs/secrets disponibles.

#### Scenario: Documentación presente en el repositorio
- **WHEN** se revisa el repositorio `.github`
- **THEN** existe `/docs/scan-sonar.md` con toda la información necesaria para que un equipo lo adopte sin asistencia adicional

---

### Requirement: Workflow template disponible
El workflow SHALL contar con un workflow template en `/workflow-templates/scan-sonar.yml` y su archivo de metadatos `/workflow-templates/scan-sonar.properties.json`, siguiendo las convenciones de GitHub.

#### Scenario: Template utilizable desde la UI de GitHub
- **WHEN** un administrador de un repositorio abre la sección "Actions" y selecciona un nuevo workflow
- **THEN** el template `scan-sonar` aparece disponible en la sección de templates de la organización
