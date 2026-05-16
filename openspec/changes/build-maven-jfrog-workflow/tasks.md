## 1. Reusable workflow Maven + JFrog

- [ ] 1.1 Crear `.github/workflows/build-maven-jfrog.yml` con `on: workflow_call`, inputs/outputs/secrets en kebab-case y permisos mínimos requeridos
- [ ] 1.2 Implementar job `build` (checkout, setup-java, cache Maven y ejecución de build con argumentos configurables)
- [ ] 1.3 Implementar job `publish` condicionado a build exitoso, con autenticación a JFrog y publicación del artefacto Maven
- [ ] 1.4 Exponer outputs de publicación en kebab-case (`artifact-coordinate`, `artifact-url`) y validar rutas de error de autenticación/publicación
- [ ] 1.5 Commit: `feat(build-maven-jfrog): crear reusable workflow de build y publicación en jfrog`

## 2. Documentación del workflow

- [ ] 2.1 Crear `docs/build-maven-jfrog.md` con propósito, repositorios/casos beneficiados y dependencias externas
- [ ] 2.2 Documentar ejemplo de consumo `uses: <org>/.github/.github/workflows/build-maven-jfrog.yml@<v1>` y comportamiento esperado por job/step
- [ ] 2.3 Incluir tablas completas de inputs, outputs y secrets (tipo, requerido, descripción y alcance repo/org para secrets)
- [ ] 2.4 Commit: `docs(build-maven-jfrog): documentar reusable workflow de maven y jfrog`

## 3. Workflow template para adopción

- [ ] 3.1 Crear `workflow-templates/build-maven-jfrog.yml` con triggers permitidos y `$default-branch` sólo en `on.push.branches` y `on.pull_request.branches`
- [ ] 3.2 Crear `workflow-templates/build-maven-jfrog.properties.json` con metadatos del template y referencia de uso del reusable workflow por tag mayor
- [ ] 3.3 Verificar que el template generado consuma `build-maven-jfrog.yml` con naming kebab-case consistente en inputs/secrets
- [ ] 3.4 Commit: `feat(build-maven-jfrog): agregar workflow template para consumo del reusable`

## 4. Validación final

- [ ] 4.1 Validar sintaxis YAML y consistencia entre workflow, documentación y template
- [ ] 4.2 Verificar escenarios críticos: build exitoso, fallo de compilación, publicación exitosa y fallo por credenciales JFrog inválidas
- [ ] 4.3 Commit: `chore(build-maven-jfrog): validar integridad del cambio openspec`
