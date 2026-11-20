# `runs` for Docker Actions

## `runs.pre-entrypoint`, `runs.entrypoint` and `runs.post-entrypoint`

|       Propiedad      |                        Función                       |
|:--------------------:|:----------------------------------------------------:|
| runs.pre-entrypoint  | Preparar el entorno antes de la ejecución principal. |
| runs.entrypoint      | Ejecutar la funcionalidad principal de la acción.    |
| runs.post-entrypoint | Realizar tareas de limpieza posteriores.             |

> `runs.post-entrypoint` no se ejecuta inmediatamente después de `runs.entrypoint`, sino durante la fase de limpieza del job, cuando ya han terminado sus pasos normales.

| Orden |              Operación              |
|:-----:|:-----------------------------------:|
| 1     | entrypoint de la acción A           |
| 2     | entrypoint de la acción B           |
| 3     | Ejecutar el comando del tercer paso |
| 4     | post-entrypoint de la acción B      |
| 5     | post-entrypoint de la acción A      |

Las acciones post se ejecutan en orden inverso al de registro (LIFO): la última acción que registra una limpieza se limpia primero.

> Por defecto, `post-entrypoint` utiliza `post-if: always()`, por lo que la limpieza está diseñada para ejecutarse incluso cuando un paso anterior haya fallado.

> Estos runs utilizan la misma imagen Docker pero, para cada uno (PRE, MAIN, POST), Github crea un contenedor diferente. Si se quiere compartir información entre ellos se puede:
> * Utilizar un workspace compartido (`$GITHUB_WORKSPACE`, directorio de trabajo del proyecto, directorio de trabajo del runner)
> * Para compartir pequeños valores Github ofrece `$GITHUB_STATE`
> * El `$HOME` entre diferentes contenedores es igual si apuntan al mismo **runner**.

# Default environment variables

`GITHUB_REF`(variable) → `${{ github.ref }}` (context property)

> You can only overwrite the value of ``CI`` (test are runned locally or in GIthub Actions?) variable. However, it`s not guaranteed that this will always be posible.

[List of DEFAULT environment variables](https://docs.github.com/en/actions/reference/workflows-and-actions/variables)

> $GITHUB_REF is only evaluated once the job is on the runner.

# Defining environment variables for a single workflow

```yaml
name: Greeting on variable day

on:
  workflow_dispatch

env:
  DAY_OF_WEEK: Monday

jobs:
  greeting_job:
    runs-on: ubuntu-latest
    env:
      Greeting: Hello
    steps:
      - name: "Say Hello Mona it's Monday"
        run: echo "$Greeting $First_Name. Today is $DAY_OF_WEEK!"
        env:
          First_Name: Mona

```

# Example: Defining outputs for a job

La clave es distinguir dos niveles de outputs:
1. Job-level outputs: permiten que un job publique valores que otros jobs pueden utilizar.
2. Workflow-level outputs: permiten que un workflow reutilizable devuelva valores al workflow que lo ha invocado.

|       Característica      |                 Job-level                |              Workflow-level              |
|:-------------------------:|:----------------------------------------:|:----------------------------------------:|
| Dónde se declara          | jobs.<job_id>.outputs                    | on.workflow_call.outputs                 |
| De dónde obtiene el valor | Habitualmente de steps.<step_id>.outputs | De jobs.<job_id>.outputs                 |
| Sintaxis                  | nombre: ${{ ... }}                       | nombre: seguido de value: ${{ ... }}     |
| Finalidad                 | Publicar resultados de un job            | Devolver resultados al workflow llamador |

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    # Map a step output to a job output
    outputs:
      output1: ${{ steps.step1.outputs.test }}
      output2: ${{ steps.step2.outputs.test }}
    steps:
      - id: step1
        run: echo "test=hello" >> "$GITHUB_OUTPUT"
      - id: step2
        run: echo "test=world" >> "$GITHUB_OUTPUT"
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - env:
          OUTPUT1: ${{needs.job1.outputs.output1}}
          OUTPUT2: ${{needs.job1.outputs.output2}}
        run: echo "$OUTPUT1 $OUTPUT2"
```

Ejemplo ChatGPT:

**Archivo A: workflow reutilizable**
``.github/workflows/calcular-version.yml``
```yaml
name: Calcular versión

on:
  workflow_call:
    outputs:
      version:
        description: "Versión calculada"
        value: ${{ jobs.calcular.outputs.version }}

jobs:
  calcular:
    runs-on: ubuntu-latest

    outputs:
      version: ${{ steps.obtener.outputs.version }}

    steps:
      - name: Obtener versión
        id: obtener
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"
```

**Archivo B: workflow llamador** ``.github/workflows/principal.yml``
```yaml
name: Workflow principal

on:
  workflow_dispatch:

jobs:
  obtener-version:
    uses: ./.github/workflows/calcular-version.yml

  mostrar-version:
    needs: obtener-version
    runs-on: ubuntu-latest

    steps:
      - name: Mostrar versión
        run: echo "Versión ${{ needs.obtener-version.outputs.version }}"
```

> `on.workflow_call.outputs` únicamente tiene sentido en un workflow que pueda ser llamado por otro workflow.

```yaml
on: push
jobs:
  generate-a-secret-output:
    runs-on: ubuntu-latest
    steps:
      - id: sets-a-secret
        name: Generate, mask, and output a secret
        run: |
          the_secret=$((RANDOM))
          echo "::add-mask::$the_secret"
          echo "secret-number=$the_secret" >> "$GITHUB_OUTPUT"
      - name: Use that secret output (protected by a mask)
        run: |
          echo "the secret number is ${{ steps.sets-a-secret.outputs.secret-number }}"

```

Ejemplo Udemy:
````yaml
name: Java CI with Maven
 
env:
  NAME: 'My Action'
 
on:
  push:
    branches: [ "main" ]
 
jobs:
  build:
    env:
        JAVA_VERSION: '11'
 
    runs-on: ubuntu-latest
 
    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK ${{env.JAVA_VERSION}}
      uses: uses: actions/setup-java@v4
      with:
        java-version: ${{env.JAVA_VERSION}}
        distribution: 'temurin'
        cache: maven
    - name: Build with Maven
      run: mvn -B package --file pom.xml
    - name: Print name
      run: echo "Hello $NAME. $BUILD. Using Java Version $JAVA_VERSION"
      env:
        BUILD: 'We are currently running the Build job'
````

# Key differences between reusable workflows and composite actions

| Reusable workflows                                                                         | Composite actions                                                                                                          |
|--------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| A YAML file, very similar to any standard workflow file                                    | An action containing a bundle of workflow steps                                                                            |
| Each reusable workflow is a single file in the .github/workflows directory of a repository | Each composite action is a separate repository, or a directory, containing an action.yml file and, optionally, other files |
| Called by referencing a specific YAML file                                                 | Called by referencing a repository or directory in which the action is defined                                             |
| Called directly within a job, not from a step                                              | Run as a step within a job                                                                                                 |
| Can contain multiple jobs                                                                  | Does not contain jobs                                                                                                      |
| Each step is logged in real-time                                                           | Logged as one step even if it contains multiple steps                                                                      |
| Can connect a maximum of ten levels of workflows                                           | Can be nested to have up to 10 composite actions in one workflow                                                           |
| Can use secrets                                                                            | Cannot use secrets                                                                                                         |
| Cannot be published to the marketplace                                                     | Can be published to the marketplace                                                                                        |

# Functions

* contains( search, item )
* startsWith( searchString, searchValue )
* endsWith( searchString, searchValue )
* format( string, replaceValue0, replaceValue1, ..., replaceValueN)
* join( array, optionalSeparator )
* toJSON(value)
* fromJSON(value)
* hashFiles(path)

If a cache key contains the dependencies file wrapped in hashFiles, the key changes when the dependencies file is updated, which helps keep it up to date.

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: ${{ runner.os }}-build-${{ env.cache-name }}-${{ hashFiles('**/package-lock.json') }}
```

# About the `GITHUB_TOKEN`
At the start of each workflow job, GitHub automatically creates a unique `GITHUB_TOKEN` secret to use in your workflow. You can use the GITHUB_TOKEN to authenticate in the workflow job.

When you enable GitHub Actions, GitHub installs a GitHub App on your repository. The `GITHUB_TOKEN` secret is a GitHub App installation access token. You can use the installation access token to authenticate on behalf of the GitHub App installed on your repository. The token's permissions are limited to the repository that contains your workflow.

> `workflow_dispatch` and `repository_dispatch` events AND pull_request events with the `opened`, `synchronize`, or `reopened` activity types always create workflow runs.

> Its permissions are limited to the repository that contains the workflow.

# ``github.context``

The github context contains information about the workflow run and the event that triggered the run.

# ``GITHUB_ENV`` Vs `outputs`

* `GITHUB_ENV` → compartir variables entre steps del mismo job. 
* `GITHUB_OUTPUT` → crear outputs de un step. 
* `jobs.<job_id>.outputs` + `needs` → compartir esos resultados entre jobs diferentes.

# [OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect)

*OpenID Connect allows your workflows to exchange short-lived tokens directly from your cloud provider.*

> Storing long-lived cloud credentials as secrets poses a risk because they must be rotated manually and remain valid even if they leak. OpenID Connect solves this by letting a workflow request a short-lived JSON Web Token from GitHub's OIDC provider, which the cloud provider validates against a trust relationship you configure and then exchanges for a short-lived cloud access token. To fetch that token, the workflow must grant the `id-token: write` permission; you can set it at the workflow level or scope it to a single job. Once configured, no cloud secret needs to be stored in GitHub, which is why the exam treats OIDC as the preferred pattern for hardening deployments.

# Github Packages

Ese permiso permite que el ``GITHUB_TOKEN`` publique paquetes, siempre que tenga acceso al registro de destino.

```yaml
permissions:
  packages: write
```
OJO es **packages**

# [REST API de GitHub](https://docs.github.com/en/rest/repos/repos?apiVersion=2026-03-10)

WORKFLOW distach event !=  REPOSITORY distach event

# [action.yml](https://docs.github.com/en/actions/reference/workflows-and-actions/metadata-syntax)

> What is the filename of the metadata file that defines the inputs, outputs, and runs configuration for your action?

## name
Required The name of your action. GitHub displays the name in the Actions tab to help visually identify actions in each job.

## author
Optional The name of the action's author.

## description
Required A short description of the action.

##  inputs
Optional Input parameters allow you to specify data that the action expects to use during runtime. GitHub stores input parameters as environment variables. We recommend using lowercase input ids.

## runs
Required Specifies whether this is a JavaScript action, a composite action, or a Docker container action and how the action is executed.

|                Propiedad               |                      ¿Para qué sirve?                     |
|:--------------------------------------:|:---------------------------------------------------------:|
| name                                   | Nombre de la acción.                                      |
| description                            | Explica qué hace.                                         |
| inputs                                 | Define los parámetros que recibe.                         |
| outputs                                | Define los resultados que puede devolver.                 |
| runs                                   | Indica cómo se ejecuta.                                   |
| runs.using                             | Especifica el tipo de ejecución, en este caso Node.js 24. |
| runs.main                              | Indica qué archivo JavaScript debe ejecutar GitHub.       |
| Can use secrets                        | Cannot use secrets                                        |
| Cannot be published to the marketplace | Can be published to the marketplace                       |


# YAML anchors and aliases

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    env: &variables_comunes
      JAVA_VERSION: '17'
      APP_ENV: production
    steps:
      - run: echo "$JAVA_VERSION"

  test:
    runs-on: ubuntu-latest
    env: *variables_comunes
    steps:
      - run: echo "$JAVA_VERSION"
```
No importa qué job se ejecute primero. Los anchors y aliases se resuelven antes de que GitHub Actions empiece a ejecutar los jobs.


# ``pull_request`` Vs `pull_request_target`

|            |                     pull_request                    |      pull_request_target      |
|:----------:|:---------------------------------------------------:|:-----------------------------:|
| GITHUB_SHA | Last merge commit on the GITHUB_REF branch          | Last commit on default branch |
| GITHUB_REF | PR merge branch refs/pull/PULL_REQUEST_NUMBER/merge | Default branch                |

# Personal Access Tokens (PATs)

* Son tokens generados **por un usuario** para autenticar de forma personalizada. 
* Tienen permisos asignados según lo que el usuario configure (lectura, escritura, administración). 
* Se usan para acceder a la API de GitHub, clonar repositorios privados desde máquinas externas, ejecutar comandos git, etc. 
* Duran mucho tiempo (hasta un año o más) o hasta que el usuario los revoque manualmente. 
* Son un tipo de credencial que representa a un usuario humano o automatización que tiene el poder de un usuario completo (según permisos asignados).

# Tokens de Instalación

* Son tokens que **GitHub genera automáticamente** para las apps instaladas (por ejemplo, GitHub Apps o runners). 
* Tienen permisos muy específicos y limitados, basados en la instalación de la app dentro de un repositorio u organización. 
* Son temporales, suelen tener una duración corta (por ejemplo, 1 hora) y se renuevan automáticamente cuando es necesario. 
* Se usan para operaciones específicas, como descargar acciones privadas desde repositorios, ejecutar tareas en los runners, etc. 
* No exponen permisos amplios ni persistentes, lo que mejora la seguridad al minimizar el riesgo si se filtran.

# Reglas de protección

* **Protección de ramas:** controla cuándo y cómo se pueden incorporar cambios al código de una rama, por ejemplo main.
> Repository → Settings → Branches → Add branch protection rule

* **Protección de despliegues:** controla cuándo y bajo qué condiciones se puede desplegar una aplicación en un entorno, por ejemplo production.
> Repository → Settings → Environments → production

Ejemplo: regla Required reviewers

# Github APIs
* Deployments API : The Deployments API is used to create and manage deployments and deployment statuses. 
* Statuses API: The Statuses API sets commit statuses from external services.
* Actions API: The Actions API in GitHub is used to interact with GitHub Actions workflows, such as triggering workflows, listing workflow runs, and getting workflow run details.
* Checks API: The Checks API is the correct choice because GitHub Actions use this API to output statuses, results, and logs for a workflow. It allows workflows to create detailed status checks, annotations, and summaries for each job and step in the workflow, providing visibility into the execution and results of the workflow.

# Permissions

## Artifacts

'write' access is not required to download workflow artifacts. Users with 'write' access can make changes to the repository, such as editing files or creating new ones, but they do not necessarily have the ability to download workflow artifacts.

```yaml
name: Descargar artifact

on:
  workflow_dispatch:

permissions:
  actions: read

jobs:
  descargar:
    runs-on: ubuntu-latest

    steps:
      - name: Descargar artifact de otra ejecución
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh run download 123456789 --name aplicacion
```

## ``GITBUB_TOKEN``

````yaml
name: Open new issue
on: workflow_dispatch

jobs:
  open-issue:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      issues: write
    steps:
      - run: |
          gh issue --repo ${{ github.repository }} \
            create --title "Issue title" --body "Issue body"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
````

#  Self hosted runners groups and labels

* Runner Groups: controlan qué repositorios pueden utilizar determinados runners. 
* Runner Labels: permiten seleccionar qué runners cumplen las características que necesita un job.

````yaml
runs-on:
      group: ubuntu-runners
      labels: ubuntu-24.04-16core
````

# ¿Qué instrucciones puedes enviar mediante workflow commands?

|          Comando         |                           Función                           |
|:------------------------:|:-----------------------------------------------------------:|
| ::warning::              | Generar una advertencia en los logs y anotaciones.          |
| ::error::                | Generar una anotación de error.                             |
| ::notice::               | Mostrar un aviso informativo.                               |
| ::debug::                | Escribir información de depuración.                         |
| ::group:: / ::endgroup:: | Agrupar líneas de logs.                                     |
| ::add-mask::             | Ocultar un valor cuando aparece posteriormente en los logs. |
| $GITHUB_ENV              | Establecer variables para los siguientes steps del job.     |
| $GITHUB_OUTPUT           | Publicar outputs de un step.                                |
| $GITHUB_PATH             | Añadir directorios al PATH de los siguientes steps.         |

# Service containers

- services: → declara los contenedores auxiliares.
- Se crean al preparar el job.
- Están disponibles para sus steps.
- Se eliminan al finalizar el job.
- Cada job dispone de sus propios service containers.
  La idea principal: un Service Container es un servicio temporal que vive lo mismo que el job que lo necesita.

# Y esta pregunta??

Suppose a workflow uses GITHUB_TOKEN to push a commit or update a pull request. Which two events will not start a new workflow run as a result? (select two)

Tu selección es incorrecta
repository_dispatch

Explicación
Incorrect. repository_dispatch explicitly allows triggering from external events.

Selección correcta
push

Explicación
Correct. Push events caused by GITHUB_TOKEN do not trigger new workflow runs to avoid recursion.
Selección correcta
pull_request

Explicación
Correct. Pull request events caused by GITHUB_TOKEN also do not trigger new runs.
Tu selección es incorrecta
workflow_dispatch

Explicación
Incorrect. workflow_dispatch explicitly allows manual triggering.


# Check later urls

* [GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
* [Available contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
* [Time limits](https://docs.github.com/en/actions/reference/limits)
* [Available contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)

* [Repo with more resources](https://github.com/LautaroOrellano/actions-certification-guide/)

```yaml
  - name: 'Upload Artifact'
    uses: actions/upload-artifact@v4
    with:
      name: my-artifact
      path: my_file.txt
      retention-days: 5
```