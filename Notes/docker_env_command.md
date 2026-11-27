Significa que **el runner Ubuntu y el contenedor Docker son dos entornos de ejecución diferentes**, cada uno con sus propias variables de entorno.

Aunque el contenedor Docker se ejecuta dentro del runner Ubuntu, no comparten necesariamente todas las variables.

## **1. Ejemplo de workflow**

```
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Mostrar variables del runner
        run: env

      - name: Ejecutar Docker Action
        uses: ./.github/actions/my-docker-action
        env:
          CUSTOM_VAR: "Hello Docker"
```

## **2. Docker Action**

**action.yml**

```
name: My Docker Action
description: Show environment variables

runs:
  using: docker
  image: Dockerfile
  env:
    INTERNAL_VAR: "Docker internal value"
```

**Dockerfile**

```
FROM alpine:3.21

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

**entrypoint.sh**

```
#!/bin/sh

echo "Variables del contenedor:"
env
```

## **3. ¿Dónde se ejecuta cada comando&#x20;**`env`**?**

**GitHub-hosted runner — Ubuntu**

**Step 1:&#x20;**`run: env`

Variables del runner Ubuntu

HOME=/home/runner

RUNNER_OS=Linux

GITHUB_WORKSPACE=/home/runner/work/...

**Contenedor Docker — Alpine**

**Step 2:&#x20;**`entrypoint.sh`**&#x20;→&#x20;**`env`

Variables del contenedor Docker

HOME=/github/home

GITHUB_WORKSPACE=/github/workspace

CUSTOM_VAR=Hello Docker

INTERNAL_VAR=Docker internal value

El contenedor tiene su propio entorno de ejecución, aunque esté alojado en el runner.

## **4. Comparación de los resultados**

| Variable           | `run: env` en el runner | `env` en `entrypoint.sh` |
| ------------------ | ----------------------- | ------------------------ |
| `HOME`             | `/home/runner`          | `/github/home`           |
| `GITHUB_WORKSPACE` | `/home/runner/work/...` | `/github/workspace`      |
| `CUSTOM_VAR`       | No disponible           | `Hello Docker`           |
| `INTERNAL_VAR`     | No disponible           | `Docker internal value`  |
| `RUNNER_OS`        | `Linux`                 | `Linux`                  |

Los valores son ilustrativos de un GitHub-hosted runner estándar.

## **5. ¿Por qué es importante?**

Si quieres comprobar que una Docker Action está recibiendo correctamente sus variables, no te sirve inspeccionar únicamente el entorno del runner.

Debes inspeccionar el entorno del propio contenedor.

En un entorno real, es preferible comprobar variables concretas con `printenv CUSTOM_VAR` en lugar de imprimirlas todas, para evitar exponer información sensible.

**Para memorizar en GH-200:**

- `run: env` → Entorno del runner, si el job no declara `container:`.
- `env` dentro de `entrypoint.sh` → Entorno de la Docker Action.
- `docker inspect` → Configuración del contenedor desde el host Docker.

El comando `env` es el mismo en ambos casos. **Lo que cambia es el entorno en el que se ejecuta.**