## 1. Reusable Workflow

- [ ] 1.1 Crear el archivo `.github/workflows/build-docker.yml` con `on: workflow_call`, definiendo inputs (`image-name`, `image-tag`, `registry-url`, `context`, `dockerfile`, `platforms`, `push`), secrets (`registry-username`, `registry-password`) y output (`image-digest`)
- [ ] 1.2 Implementar el job `build` con los steps: checkout (`actions/checkout@v4`), setup QEMU (`docker/setup-qemu-action@v3`), setup Buildx (`docker/setup-buildx-action@v3`), login al registry (`docker/login-action@v3`) y build+push (`docker/build-push-action@v6`)
- [ ] 1.3 Mapear correctamente los inputs y secrets a cada step y capturar el output `image-digest` desde el step de build+push

## 2. Documentación

- [ ] 2.1 Crear el archivo `/docs/build-docker.md` con: propósito del workflow, ejemplo de uso completo con `uses`, `with` y `secrets`, tabla de inputs (nombre, tipo, requerido, default, descripción), tabla de outputs y tabla de secrets
- [ ] 2.2 Agregar sección de notas de versionado indicando que el `uses` debe apuntar siempre al último tag semántico (`@vX`)

## 3. Workflow Template

- [ ] 3.1 Crear `/workflow-templates/build-docker.yml` con un ejemplo de uso del reusable workflow apuntando al tag semántico actual (`@v1`), usando `$default-branch` sólo en triggers `on.push.branches` y `on.pull_request.branches`
- [ ] 3.2 Crear `/workflow-templates/build-docker.properties.json` con los metadatos requeridos por GitHub: `name`, `description`, `iconName`, `categories`
