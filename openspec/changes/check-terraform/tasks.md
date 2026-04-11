## 1. Reusable Workflow

- [ ] 1.1 Crear `.github/workflows/check-terraform.yml` con el job `validate`, inputs en kebab-case (`terraform-version`, `working-directory`), steps: checkout con `actions/checkout@v4`, setup con `hashicorp/setup-terraform@v3`, `terraform fmt -check -recursive`, `terraform init -backend=false` y `terraform validate`. Sin secrets ni autenticación de proveedor.
  Commit: `feat(check-terraform): agregar reusable workflow de check de formato y validación de terraform`

## 2. Documentación

- [ ] 2.1 Crear `docs/check-terraform.md` con la descripción del propósito del workflow (agnóstico al proveedor), ejemplo completo de consumo con `uses:`, tabla de inputs con tipo/requerido/default/descripción, aclaración de que no requiere secrets, y descripción del job `validate` con sus steps principales.
  Commit: `docs(check-terraform): agregar documentación del reusable workflow`

## 3. Workflow Template

- [ ] 3.1 Crear `workflow-templates/check-terraform.yml` con la referencia al reusable workflow apuntando al último tag, y `workflow-templates/check-terraform.properties.json` con los metadatos del template.
  Commit: `feat(check-terraform): agregar workflow template para nuevos repositorios`
