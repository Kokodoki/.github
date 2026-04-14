## Context

Los repositorios de la organización carecen de un mecanismo centralizado de análisis de calidad de código. SonarCloud es la plataforma elegida por el equipo para análisis estático; actualmente cada equipo lo integra ad-hoc o directamente no lo integra. Un reusable workflow en el repositorio `.github` permite estandarizar la integración con mínimo esfuerzo por parte de los consumidores.

El workflow se implementa en `.github/workflows/scan-sonar.yml` usando `on: workflow_call` y la action oficial `SonarSource/sonarcloud-github-action`. Se acompaña de documentación en `/docs/scan-sonar.md` y de un workflow template en `/workflow-templates/`.

## Goals / Non-Goals

**Goals:**
- Centralizar la configuración del scanner de SonarCloud en un único reusable workflow.
- Validar el Quality Gate de SonarCloud y fallar el pipeline si no pasa.
- Ser consumible desde cualquier repositorio de la organización con la mínima configuración posible.
- Seguir las convenciones del repositorio: nomenclatura kebab-case, documentación en `/docs/`, workflow template.

**Non-Goals:**
- Instalar o configurar herramientas de build del proyecto consumidor (ej. Maven, Gradle, npm) — responsabilidad del caller.
- Gestionar tokens de SonarCloud ni crearlos automáticamente.
- Soportar self-hosted SonarQube (solo SonarCloud SaaS).
- Publicar resultados en comentarios de PR (puede lograrse con configuración adicional del `sonar-project.properties` del consumidor).

## Decisions

### D1 — Usar `SonarSource/sonarcloud-github-action` como action principal

**Decisión**: Usar la action oficial `SonarSource/sonarcloud-github-action@v3` en lugar de instalar el CLI manualmente.

**Rationale**: La action oficial encapsula la instalación del scanner, la detección del lenguaje y la integración con las Pull Requests de GitHub de forma automática. Reduce el mantenimiento del workflow.

**Alternativa descartada**: Instalar `sonar-scanner` CLI directamente con un step de `run`. Más flexible pero requiere mantener la versión del CLI y la configuración manual.

---

### D2 — El Quality Gate se valida a través del parámetro `scannerMode` + `waitForQualityGate`

**Decisión**: La action `SonarSource/sonarcloud-github-action@v3` con `args: "-Dsonar.qualitygate.wait=true"` bloquea el job hasta que el Quality Gate sea evaluado y falla si no pasa.

**Rationale**: Es la forma recomendada por SonarSource para bloquear pipelines en función del Quality Gate sin un step adicional.

**Alternativa descartada**: Llamar a la API REST de SonarCloud en un step separado para verificar el Quality Gate. Mayor complejidad y requiere lógica adicional.

---

### D3 — Inputs mínimos y opcionales

**Decisión**: Solo `sonar-project-key` y `sonar-organization` son requeridos. El resto (`main-branch`, `extra-args`) son opcionales con defaults razonables.

**Rationale**: Minimiza la fricción de adopción. La mayoría de proyectos solo necesitan la clave y la organización.

---

### D4 — El tag del reusable workflow en el template apunta al último tag mayor

**Decisión**: El workflow template usa `uses: Kokodoki/.github/.github/workflows/scan-sonar.yml@v1`.

**Rationale**: Convención del repositorio — nunca apuntar a un branch ni a un SHA.

## Risks / Trade-offs

- **[Riesgo] SonarCloud no está disponible** → El job falla y bloquea el pipeline. Mitigación: documentar que el consumidor puede usar `continue-on-error: true` si el análisis es no-bloqueante en su contexto.
- **[Riesgo] `sonar.qualitygate.wait=true` incrementa el tiempo de pipeline** → El job espera hasta que SonarCloud finalice el análisis (puede tardar 1-5 minutos). Mitigación: documentarlo y permitir que el consumidor lo desactive vía `extra-args`.
- **[Trade-off] Sin outputs** → El workflow no expone el estado del Quality Gate como output de GitHub Actions. Se decidió no hacerlo para mantener la simplicidad; el fallo del job es señal suficiente.
