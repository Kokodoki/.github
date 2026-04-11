## Why

Los equipos de la organización gestionan infraestructura con Terraform usando diferentes proveedores cloud (AWS, Azure, GCP, etc.). Actualmente no existe un mecanismo centralizado y agnóstico al proveedor que valide el formato y la sintaxis de los archivos `.tf` antes de ejecutar un plan o apply. Esto provoca que errores de formato o configuración lleguen a etapas más avanzadas del pipeline, aumentando el tiempo de feedback y el riesgo de fallos en producción.

## What Changes

- Se introduce el reusable workflow `check-terraform.yml` en `.github/workflows/` con un único job `validate` que ejecuta secuencialmente `terraform fmt -check -recursive` y `terraform validate` con `terraform init -backend=false`, de forma completamente agnóstica al proveedor cloud.
- No se requiere ningún secret ni credencial de proveedor: el workflow sólo verifica formato y sintaxis local.
- Se crea la documentación del workflow en `docs/check-terraform.md`.
- Se crea el workflow template en `workflow-templates/check-terraform.yml` y `workflow-templates/check-terraform.properties.json`.

## Capabilities

### New Capabilities

- `check-terraform`: Reusable workflow que ejecuta un check de formato (`terraform fmt -check -recursive`) y validación de sintaxis (`terraform validate` con backend deshabilitado) sobre el directorio indicado por `working-directory`, sin requerir credenciales de ningún proveedor cloud.

### Modified Capabilities

## Impact

- **Inputs requeridos:** ninguno
- **Inputs opcionales:** `terraform-version`, `working-directory`
- **Outputs:** ninguno
- **Secrets requeridos:** ninguno
- **Dependencias externas:** `actions/checkout@v6`, `hashicorp/setup-terraform@v4`
- **Repositorios beneficiados:** cualquier repositorio de la organización que gestione infraestructura con Terraform, independientemente del proveedor cloud utilizado
- **Requisito previo:** los módulos o providers referenciados deben poder descargarse (o el directorio debe ser auto-suficiente), aunque el backend no se inicializa
