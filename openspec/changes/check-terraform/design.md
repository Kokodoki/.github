## Context

La organización dispone de múltiples repositorios que gestionan infraestructura con Terraform sobre distintos proveedores cloud. No existe un estándar centralizado para verificar el formato y la validez sintáctica de los archivos `.tf` antes de ejecutar etapas costosas como `plan` o `apply`. El workflow `check-terraform.yml` cubre exclusivamente esta etapa de validación ligera, sin necesidad de credenciales ni backend remoto.

## Goals / Non-Goals

**Goals:**

- Verificar el formato de los archivos `.tf` con `terraform fmt -check -recursive`.
- Validar la sintaxis y consistencia de la configuración con `terraform validate`, usando `terraform init -backend=false` para evitar la necesidad de credenciales o backend remoto.
- Ser completamente agnóstico al proveedor cloud (AWS, Azure, GCP, etc.).
- No requerir ningún secret ni credencial de proveedor.
- Exponer inputs en kebab-case.
- Acompañar el workflow con documentación en `docs/check-terraform.md` y con workflow template en `workflow-templates/`.

**Non-Goals:**

- Ejecutar `terraform plan` o `terraform apply`.
- Gestionar el estado remoto de Terraform.
- Soportar autenticación con ningún proveedor cloud.
- Ejecutar linting adicional (e.g. tflint, tfsec, checkov).

## Decisions

### 1. Un único job `validate` con responsabilidad única

**Decisión:** El workflow contiene un único job `validate` con tres steps: checkout, setup-terraform y la secuencia `fmt → init -backend=false → validate`.

**Rationale:** La validación de formato y sintaxis es una operación liviana que no requiere separación en jobs. Un único job simplifica el workflow y reduce el tiempo de ejecución.

**Alternativa descartada:** Separar `fmt` y `validate` en dos jobs distintos — añade complejidad innecesaria para operaciones tan ligeras.

### 2. `terraform init -backend=false` antes de `validate`

**Decisión:** Se ejecuta `terraform init -backend=false` como pre-requisito de `terraform validate`.

**Rationale:** `terraform validate` requiere que los providers estén disponibles en el directorio `.terraform` para poder resolver los tipos de recursos. Con `-backend=false` se descargan los providers sin necesitar credenciales ni configuración de backend remoto, manteniendo el workflow agnóstico al proveedor.

**Alternativa descartada:** Saltar el init completamente — `terraform validate` falla si los providers no están inicializados en módulos complejos.

### 3. Agnóstico al proveedor mediante ausencia de steps de autenticación

**Decisión:** El workflow no incluye ningún step de autenticación con proveedores cloud.

**Rationale:** El objetivo es únicamente validar formato y sintaxis local. Cualquier autenticación sería innecesaria y acoplaría el workflow a un proveedor específico.

### 4. Inputs opcionales con defaults razonables

**Decisión:** Ambos inputs (`terraform-version` y `working-directory`) son opcionales con defaults `latest` y `.` respectivamente.

**Rationale:** La mayoría de los repositorios pueden usar la versión más reciente de Terraform y trabajar desde la raíz del repositorio, reduciendo la configuración necesaria para los consumidores.

## Estructura de archivos

```
.github/workflows/check-terraform.yml              # Reusable workflow
docs/check-terraform.md                            # Documentación
workflow-templates/check-terraform.yml             # Template para nuevos repos
workflow-templates/check-terraform.properties.json # Metadatos del template
```

### Inputs del workflow

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos .tf |

### Secrets del workflow

Ninguno. El workflow no requiere credenciales de ningún proveedor.

### Outputs del workflow

Ninguno.

## Estructura del job `validate`

| Step | Comando | Descripción |
|---|---|---|
| Checkout | `actions/checkout@v6` | Descarga el código del repositorio consumidor |
| Setup Terraform | `hashicorp/setup-terraform@v4` | Instala la versión indicada de Terraform |
| Format check | `terraform fmt -check -recursive` | Falla si algún archivo `.tf` no cumple el formato estándar |
| Init (sin backend) | `terraform init -backend=false` | Descarga providers localmente sin inicializar el backend remoto |
| Validate | `terraform validate` | Valida la sintaxis y consistencia de la configuración |

## Risks / Trade-offs

- **Providers que requieren credenciales en `init`** → Algunos providers pueden intentar validar credenciales durante el init incluso con `-backend=false`. Mitigation: documentar que el workflow asume que los providers son descargables sin autenticación, o que el repositorio consumidor puede sobrescribir la versión de Terraform si usa providers que lo requieren.
- **Módulos locales con referencias externas** → Si el código Terraform referencia módulos externos que no están en el repositorio, el `init -backend=false` los descargará de sus fuentes (registry, git). Mitigation: esto es comportamiento esperado y no requiere credenciales de proveedor cloud.
- **Versión `latest` de Terraform** → Usar `latest` puede introducir cambios de comportamiento entre ejecuciones. Mitigation: documentar que se recomienda fijar la versión en repositorios de producción.
