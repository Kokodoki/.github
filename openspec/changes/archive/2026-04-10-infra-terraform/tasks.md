## 1. Reusable Workflow

- [x] 1.1 Crear `.github/workflows/infra-terraform.yml` con los tres jobs (`validate`, `plan`, `apply`), inputs en kebab-case (`terraform-version`, `working-directory`, `aws-region`, `environment-name`, `backend-config`), input `aws-role-to-assume` y secrets `aws-access-key-id` y `aws-secret-access-key` (OIDC o credenciales estáticas mutuamente excluyentes), output `plan-exit-code`, paso del tfplan mediante `actions/upload-artifact@v4` y `actions/download-artifact@v4`, comentario del plan en PRs con `actions/github-script@v7`, y gate de aprobación mediante `environment:` en el job `apply`.
  Commit: `feat(infra-terraform): agregar reusable workflow de terraform con jobs validate, plan y apply`

## 2. Documentación

- [x] 2.1 Crear `docs/infra-terraform.md` con la descripción del propósito del workflow, ejemplo completo de consumo con `uses:`, tabla de inputs con tipo/requerido/default/descripción, tabla de secrets, tabla de outputs, y descripción de cada job con sus steps principales.
  Commit: `docs(infra-terraform): agregar documentación del reusable workflow`

## 3. Workflow Template

- [x] 3.1 Crear `workflow-templates/infra-terraform.yml` y `workflow-templates/infra-terraform.properties.json`
  Commit: `feat(infra-terraform): agregar workflow template para nuevos repositorios`
