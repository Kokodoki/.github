## 1. Reusable Workflow

- [ ] 1.1 Crear `.github/workflows/scan-sonar.yml` con `on: workflow_call`, inputs (`sonar-project-key`, `sonar-organization`, `main-branch`, `extra-args`), secret `sonar-token` y el job que ejecuta `SonarSource/sonarcloud-github-action@v3` con `sonar.qualitygate.wait=true`

## 2. Documentación

- [ ] 2.1 Crear `/docs/scan-sonar.md` con propósito del workflow, ejemplo de uso completo, descripción del job y sus steps, tabla de inputs y secrets

## 3. Workflow Template

- [ ] 3.1 Crear `/workflow-templates/scan-sonar.yml` con el template de uso del reusable workflow apuntando a `Kokodoki/.github/.github/workflows/scan-sonar.yml@v1`
- [ ] 3.2 Crear `/workflow-templates/scan-sonar.properties.json` con los metadatos del template (nombre, descripción, categorías, iconName)
