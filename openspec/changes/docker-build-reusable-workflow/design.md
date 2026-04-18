## Context

El repositorio `.github` de la organización centraliza reusable workflows de GitHub Actions. Actualmente no existe un workflow para Docker builds, por lo que cada repositorio duplica esta lógica. Se introduce `build-docker.yml` siguiendo las mismas convenciones que `infra-terraform.yml` ya existente.

El workflow debe ser consumido con la sintaxis `uses: <org>/.github/.github/workflows/build-docker.yml@<vX>`, apuntando siempre a un tag semántico.

## Goals / Non-Goals

**Goals:**
- Proveer un reusable workflow `build-docker.yml` con `on: workflow_call` que encapsule checkout, configuración de QEMU y Buildx, autenticación en registry y build+push de la imagen.
- Exponer inputs bien definidos en kebab-case (imagen, tag, contexto, Dockerfile, plataforma, push habilitado, registry URL).
- Exponer outputs: `image-digest` con el digest SHA de la imagen construida.
- Proveer secrets en kebab-case: `registry-username` y `registry-password`.
- Acompañar con documentación en `/docs/build-docker.md` y workflow template en `/workflow-templates/`.

**Non-Goals:**
- Escaneo de vulnerabilidades de la imagen (scope de otro workflow).
- Publicación a múltiples registries en paralelo en una sola llamada.
- Gestión de Dockerfile: el consumidor provee el Dockerfile listo.

## Decisions

### Usar `docker/build-push-action` como action principal
**Rationale**: Es la action oficial de Docker mantenida por Docker Inc., ampliamente adoptada, soporta Buildx y multi-plataforma nativamente.  
**Alternativa considerada**: Ejecutar comandos `docker build` directamente via `run`. Se descartó por mayor complejidad de scripting y falta de mantenimiento centralizado.

### Separar setup de QEMU y Buildx en steps explícitos
**Rationale**: Permite habilitar/deshabilitar soporte multi-arch con claridad. QEMU sólo es necesario para builds cross-platform; mantenerlo separado es más legible.

### Input `push` de tipo boolean, `true` por defecto
**Rationale**: Permite usar el mismo workflow para builds de verificación (PR, rama feature) sin publicar la imagen, simplemente pasando `push: false`.

### Versiones de actions con tag flotante mayor
**Rationale**: La convención del repositorio exige referenciar el tag mayor flotante (e.g., `@v6`, `@v3`) en lugar de SHAs o branches, para recibir parches automáticos de seguridad sin romper compatibilidad.

### Tag del workflow template apunta al último tag de release
**Rationale**: Los workflow templates deben apuntar a un tag de release estable, no a un branch, para garantizar reproducibilidad en los repositorios consumidores.

## Risks / Trade-offs

- **[Riesgo] Cambio de API en actions de terceros** → Mitigación: usar tags flotantes mayores que reciben sólo parches compatibles. Revisar changelogs al actualizar versión mayor.
- **[Trade-off] Input `push: true` por defecto** → Si un consumidor olvida sobreescribir, podría publicar imágenes no deseadas. Mitigación: documentar claramente y recomendar `push: false` en contextos de PR.
- **[Riesgo] Secrets no pasados correctamente** → El build fallará con error de autenticación. Mitigación: validar en docs que los secrets deben pasarse explícitamente (`secrets: inherit` o explícitos).
