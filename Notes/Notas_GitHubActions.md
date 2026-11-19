    * [Reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
* [Permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions)
* `repository_dispatch` Vs `workflow_dispatch`. [Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)

> Este evento permite activar un workflow de GitHub Actions mediante una solicitud HTTP POST a la API de GitHub, normalmente desde una aplicación o servicio externo al repositorio.
> a manually trigger a workflow from the GitHub UI, será workflow_dispatch.


+ 

```yaml
jobs:
  ReusableMatrixJobForDeployment:
    strategy:
      matrix:
        target: [dev, stage, prod]

    uses: octocat/octo-repo/.github/workflows/deployment.yml@main

    with:
      target: ${{ matrix.target }}
```
| Ejecución |   Workflow llamado  | Valor enviado |
|:---------:|:-------------------:|:-------------:|
| 1         | deployment.yml@main | target: dev   |
| 2         | deployment.yml@main | target: stage |
| 3         | deployment.yml@main | target: prod  |

```yaml
name: Deployment

on:
  workflow_call:
    inputs:
      target: # Coger datos del campo target
        description: "Entorno de despliegue"
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Mostrar entorno
        env:
          TARGET: ${{ inputs.target }}
        run: echo "Desplegando en $TARGET"
```

> ¿Y si quiero que se ejecuten uno detrás de otro?
```yaml
   strategy:
   max-parallel: 1
   matrix:
   target: [dev, stage, prod]
```

Esto limita la ejecución a una combinación de la matriz a la vez, pero no garantiza contractualmente el orden dev → stage → prod.

* [Trigger a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)

> No se pueden utilizar `braches` y `branches_ignore` a la vez para el mismo **EVENTO** en un **workflow**.
> Lo mismo con tags.

Supongamos que quieres ejecutar las pruebas únicamente cuando se modifiquen archivos Java de la rama main.
```yaml
on:
  push:
    branches:
      - main
    paths:
      - '**/*.java'
```

|   Rama  |   Archivos modificados   | ¿Se ejecuta? |
|:-------:|:------------------------:|:------------:|
| main    | src/App.java             | Sí           |
| main    | README.md                | No           |
| develop | src/App.java             | No           |
| develop | README.md                | No           |
| main    | README.md y src/App.java | Sí           |

> [!tip]
> GitHub Actions utiliza patrones **glob** para filtrar ramas y etiquetas.

# workflow_call VS workflow_run

| Característica |                   workflow_call                   |                           workflow_run                           |
|:--------------:|:-------------------------------------------------:|:----------------------------------------------------------------:|
| Finalidad      | Reutilizar un workflow                            | Reaccionar a la ejecución de otro workflow                       |
| Cómo se activa | Mediante jobs.<job_id>.uses                       | Mediante el evento workflow_run                                  |
| Relación       | Llamador → llamado                                | Workflow A desencadena B                                         |
| Parámetros     | inputs y secrets                                  | Datos del evento mediante github.event.workflow_run              |
| Ejecución      | Dentro de la ejecución del workflow llamador      | Nueva ejecución independiente                                    |
| Dependencias   | El llamador espera al reutilizable                | No se establece una dependencia needs                            |
| Permisos       | El llamado no puede elevar los permisos recibidos | El segundo workflow puede disponer de permisos y secrets propios |
| Uso habitual   | Reutilizar compilaciones, pruebas o despliegues   | Lanzar procesos después de otros workflows                       |

> Con workflow_call, el workflow reutilizable no puede elevar los permisos del GITHUB_TOKEN que recibe del llamador.

Con workflow_run, en cambio, el segundo workflow es independiente y puede tener acceso a sus propios secretos y permisos de escritura, aunque el primero no los tuviera. Esto exige especial cuidado cuando el primer workflow procesa código no confiable.

|                               Necesidad                              |    Utilizar   |
|:--------------------------------------------------------------------:|:-------------:|
| Reutilizar el mismo procedimiento de despliegue en diez repositorios | workflow_call |
| Pasar parámetros a un workflow reutilizable                          | workflow_call |
| Ejecutar un workflow de despliegue cuando termine otro de CI         | workflow_run  |
| Ejecutar un workflow cuando otro comience                            | workflow_run  |
| Centralizar un proceso que contiene varios jobs                      | workflow_call |

# build zip ejemplo

```yaml
name: Build

on:
  workflow_call:
    outputs:
      zip_path:
        description: "Ruta del archivo ZIP"
        value: ${{ jobs.build.outputs.zip_path }}  # NIVEL 3

jobs:
  build:
    runs-on: ubuntu-latest

    outputs:
      zip_path: ${{ steps.package.outputs.zip_path }}  # NIVEL 2

    steps:
      - name: Crear ZIP
        id: package
        run: |
          mkdir -p dist
          zip dist/app.zip README.md
          echo "zip_path=dist/app.zip" >> "$GITHUB_OUTPUT"  # NIVEL 1
```

# Default settings

When more than one default setting is defined with the same name, GitHub uses the most specific default setting. In other words, worflow < job < step.

Un último detalle: defaults.run solo afecta a los pasos que utilizan ``run``, no a los pasos que invocan acciones mediante uses. Además, los directorios deben existir cuando se ejecute el comando.

# Status check functions

* success
* always -> Causes the step to always execute, and returns true, even when canceled. The always expression is best used at the step level or on tasks that you expect to run even when a job is canceled. For example, you can use always to send logs even when a job is canceled.
* cancelled
* failure

# jobs.<job_id>.steps[*]
* continue-on-error
* timeout-minutes
* background
* wait (other steps) `wait: [build-frontend, build-backend]`
* wait-all
* cancel
* parallel (all of steps in parrallel have to finish in their entirety)

# GIT_OUPUT Vs GIT_ENV
https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands#setting-an-environment-variable

# ¿Qué ocurre con la caché creada por una Pull Request?
   Esta es probablemente la parte más importante para el examen GH-200.

Cuando un workflow se ejecuta mediante el evento pull_request, GitHub utiliza una referencia especial:
refs/pull/25/merge


Supongamos que tenemos la Pull Request #25, que propone fusionar feature-b en feature-a.

Si el workflow crea una caché, esta queda asociada a refs/pull/25/merge, en lugar de convertirse en una caché general de feature-b.

Esto tiene una consecuencia

|           Ejecución           | ¿Puede restaurar la caché de PR #25? |
|:-----------------------------:|:------------------------------------:|
| Otra ejecución de la PR #25   | Sí                                   |
| Una ejecución desde feature-a | No                                   |
| Una ejecución desde main      | No                                   |
| Una ejecución de la PR #26    | No                                   |

>Por tanto, las cachés creadas por una Pull Request quedan restringidas a esa misma Pull Request.

# actions/download Vs actions/upload

# Deployment environments

Environments are used to describe a general deployment target like production, staging, or development. When a GitHub Actions workflow deploys to an environment, the environment is displayed on the main page of the repository. You can use environments to require approval for a job to proceed, restrict which branches can trigger a workflow, gate deployments with custom deployment protection rules, or limit access to secrets. For more information about creating environments, see Managing environments for deployment.

**Each job in a workflow can reference a single environment.** Any protection rules configured for the environment must pass before a job referencing the environment is sent to a runner. The job can access the environment's secrets only after the job is sent to a runner.

When a workflow references an environment, the environment will appear in the repository's deployments. For more information about viewing current and previous deployments, see Viewing deployment history.

# vARIABLES 

|      Variable     |                                  Contenido                                  |
|:-----------------:|:---------------------------------------------------------------------------:|
| GITHUB_REPOSITORY | Nombre del repositorio con su propietario, por ejemplo octocat/hello-world. |
| GITHUB_ACTOR      | Usuario o aplicación que inició la ejecución, por ejemplo octocat.          |
| GITHUB_WORKFLOW   | Nombre del workflow que se está ejecutando.                                 |

# Prioridad de los `secretos` en GitHub Actions
Los secretos pueden definirse en tres niveles:
PRIORIDAD MÁS ALTA

1. Environment

Secreto específico de un entorno


2. Repository

Secreto del repositorio


PRIORIDAD MÁS BAJA

3. Organization

Secreto compartido por la organización


# Example: Setting a debug message
```bash
echo "::debug::Set the Octocat variable"
```

# Github Self Hosted Runners

https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/monitor-and-troubleshoot

* --check: comprueba la conectividad con los servicios de GitHub y genera resultados de diagnóstico.
* --diag: genera información de diagnóstico del runner, pero no realiza específicamente la comprobación de conectividad.

# reutilización dentro de una organización

* Cache: puede reutilizarse entre workflows del mismo repositorio, pero no se comparte directamente entre repositorios diferentes.
* Artifacts: pueden descargarse desde distintas ejecuciones y workflows, con los permisos adecuados, pero pertenecen a ejecuciones concretas, no a una configuración compartida de organización.



# Expanding or adding matrix configurations
To expand existing matrix configurations or to add new configurations, use jobs.<job_id>.strategy.matrix.include. The value of include is a list of objects.

For example, consider the following matrix.

```yaml
strategy:
  matrix:
    fruit: [apple, pear]
    animal: [cat, dog]
    include:
      - color: green
      - color: pink
        animal: cat
      - fruit: apple
        shape: circle
      - fruit: banana
      - fruit: banana
        animal: cat
```

This will result in six jobs with the following matrix combinations.

* {fruit: apple, animal: cat, color: pink, shape: circle}
* {fruit: apple, animal: dog, color: green, shape: circle} 
* {fruit: pear, animal: cat, color: pink} 
* {fruit: pear, animal: dog, color: green} 
* {fruit: banana} 
* {fruit: banana, animal: cat}

# Example of a multiline string
This example uses EOF as the delimiter, and sets the JSON_RESPONSE environment variable to the value of the curl response.

```yaml
steps:
- name: Set the value in bash
  id: step_one
  run: |
  {
  echo 'JSON_RESPONSE<<EOF'
  curl https://example.com
  echo EOF
  } >> "$GITHUB_ENV"
```
