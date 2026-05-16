## Context

La organización necesita un reusable workflow estandarizado para proyectos Java con Maven que compile y publique artefactos en JFrog Artifactory. Actualmente cada repositorio implementa pipelines distintos, con variaciones en autenticación, parámetros de build y publicación.

El diseño debe alinearse con las convenciones del repositorio `.github`: archivo de workflow reusable, documentación en `docs/` y workflow template en `workflow-templates/`, usando nombres en kebab-case para inputs/outputs/secrets.

## Goals / Non-Goals

**Goals:**
- Estandarizar un workflow reusable `build-maven-jfrog.yml` para compilar y publicar artefactos Maven en JFrog.
- Definir interfaz consistente de `workflow_call` con inputs, outputs y secrets en kebab-case.
- Separar responsabilidades por jobs (build/publicación) y generar outputs útiles para consumidores.
- Incluir documentación y workflow template listos para adopción en nuevos repositorios.
- Referenciar actions con tags flotantes mayores y mantener compatibilidad con consumo vía tag del reusable workflow.

**Non-Goals:**
- Gestionar promoción avanzada de artefactos entre repositorios de JFrog (dev/stage/prod).
- Implementar escaneo de seguridad o quality gates adicionales fuera del build/publicación base.
- Soportar herramientas de build distintas de Maven (Gradle, Ant).

## Decisions

1. **Interfaz de entrada estándar vía `workflow_call`:**
   Se definirán inputs para versión de Java, versión de Maven, `working-directory`, argumentos de build y coordenadas/repositorio objetivo en JFrog.
   - Alternativa considerada: hardcodear versiones y paths.
   - Razón de descarte: reduce reutilización entre repositorios con necesidades distintas.

2. **Separación en jobs con responsabilidad única:**
   - `build`: checkout, setup-java, cache Maven y ejecución de `mvn`.
   - `publish`: autenticación en JFrog y publicación del artefacto Maven.
   - Alternativa considerada: un único job monolítico.
   - Razón de descarte: menor trazabilidad, menor posibilidad de extender/reutilizar y peor diagnóstico de fallas.

3. **Autenticación por secrets explícitos y alcance configurable:**
   Se usarán secrets en kebab-case para `jfrog-url`, `jfrog-user` y `jfrog-token` (o equivalente API key), documentando si deben definirse a nivel repositorio u organización.
   - Alternativa considerada: credenciales embebidas por variable global fija.
   - Razón de descarte: menor seguridad y menor portabilidad.

4. **Outputs para trazabilidad de publicación:**
   Se expondrán outputs con información del artefacto publicado (por ejemplo, `artifact-coordinate` y `artifact-url`) para integraciones posteriores.
   - Alternativa considerada: sin outputs.
   - Razón de descarte: limita orquestación de flujos downstream.

5. **Plantilla de workflow para adopción rápida:**
   Se incluirá `workflow-templates/build-maven-jfrog.yml` + `.properties.json`, usando `$default-branch` sólo en triggers de push/pull_request y `uses` apuntando a tag mayor del reusable.
   - Alternativa considerada: solo documentación textual.
   - Razón de descarte: mayor fricción al onboarding.

## Risks / Trade-offs

- **[Riesgo] Variabilidad en configuración de repositorios Maven/JFrog por proyecto** → **Mitigación:** inputs obligatorios y opcionales bien documentados, con validaciones tempranas.
- **[Riesgo] Fallas de publicación por credenciales inválidas o permisos insuficientes** → **Mitigación:** mensajes de error claros, fail-fast en autenticación y guía de secrets en documentación.
- **[Riesgo] Incremento de tiempo de pipeline por descargas Maven** → **Mitigación:** habilitar cache de dependencias Maven.
- **[Trade-off] Flexibilidad vs simplicidad de la interfaz** → **Mitigación:** definir un set mínimo de inputs obligatorios y ampliar opcionales solo cuando aporten valor común.

## Migration Plan

1. Implementar el reusable workflow en `.github/workflows/build-maven-jfrog.yml`.
2. Crear documentación `docs/build-maven-jfrog.md` con ejemplo de consumo e interfaz completa.
3. Crear template en `workflow-templates/` que consuma el reusable mediante tag mayor.
4. Validar sintaxis y uso en un repositorio piloto Java.
5. Publicar versión etiquetada del repositorio `.github` para consumo organizacional.
6. Rollback: volver a tag mayor anterior en los repositorios consumidores si se detecta regresión.

## Open Questions

- ¿Se estandarizará un repositorio JFrog por defecto (por ejemplo, `maven-release-local`) o será siempre input obligatorio?
- ¿La publicación debe ejecutarse sólo en `main`/tags o también en branches de feature bajo una bandera?
- ¿Se requiere soportar snapshot y release con rutas/credenciales separadas desde la primera versión?
