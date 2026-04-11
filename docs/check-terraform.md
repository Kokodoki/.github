# check-terraform

Reusable workflow que verifica el formato y la validez sintáctica de los archivos Terraform de forma completamente agnóstica al proveedor cloud. No requiere credenciales ni inicialización del backend remoto.

## Cómo consumirlo

```yaml
name: Check Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    uses: Kokodoki/.github/.github/workflows/check-terraform.yml@<v1>
    with:
      working-directory: infra/
      terraform-version: "1.9.0"
```

## Inputs

| Input | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `terraform-version` | string | No | `latest` | Versión de Terraform a utilizar |
| `working-directory` | string | No | `.` | Directorio raíz de los archivos `.tf` |

## Secrets

Ninguno. El workflow no requiere credenciales de ningún proveedor cloud.

## Outputs

Ninguno.

## Permissions requeridos

```yaml
permissions:
  contents: read
```

## Jobs

### `validate`

Ejecuta validaciones estáticas sin necesidad de backend remoto ni credenciales de proveedor.

| Step | Descripción |
|---|---|
| Checkout | Descarga el código del repositorio consumidor |
| Setup Terraform | Instala la versión especificada de Terraform |
| Terraform Format | Verifica el formato con `terraform fmt -check -recursive`. Falla si hay archivos mal formateados |
| Terraform Init (sin backend) | Inicializa el directorio de trabajo con `terraform init -backend=false` para resolver providers sin conectar al backend remoto |
| Terraform Validate | Valida la sintaxis y coherencia de los archivos `.tf` con `terraform validate` |

## Notas

- Se recomienda fijar `terraform-version` en repositorios de producción para evitar cambios de comportamiento entre ejecuciones.
- El workflow asume que los providers referenciados son descargables públicamente. Si un provider requiere autenticación para su descarga, deberá usarse un workflow más específico.
- Los módulos externos (registry, git) se descargan durante el `init -backend=false`, lo cual es comportamiento esperado y no requiere credenciales de proveedor cloud.
