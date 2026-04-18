## 1. Reusable Workflow

- [ ] 1.1 Crear `.github/workflows/build-docker.yml` con el job `build` que incluye autenticación AWS (OIDC o credenciales estáticas), login a ECR y build+push de la imagen Docker

## 2. Documentación

- [ ] 2.1 Crear `docs/build-docker.md` con propósito, tabla de inputs/outputs/secrets, descripción del job `build` y ejemplo de uso

## 3. Workflow Template

- [ ] 3.1 Crear `workflow-templates/build-docker.yml` con el template para que los consumidores puedan adoptar el workflow desde la UI de GitHub
- [ ] 3.2 Crear `workflow-templates/build-docker.properties.json` con los metadatos del template (nombre, descripción, categorías)
