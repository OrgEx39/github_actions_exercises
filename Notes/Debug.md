Para activar el **debug en el workflow de GitHub Actions** que te pasé (`main.yml` y `reusable.yml`), tienes dos opciones.

## **1. Activar el debug desde el propio YAML**

Añade estas variables al workflow principal (`main.yml`):

```
name: Workflow principal

on:
  workflow_dispatch:

env:
  ACTIONS_STEP_DEBUG: "true"
  ACTIONS_RUNNER_DEBUG: "true"

jobs:
  llamar-reusable:
    uses: ./.github/workflows/reusable.yml

    with:
      environment: production
      version: "2.5.0"

    secrets:
      API_TOKEN: ${{ secrets.MY_API_TOKEN }}
```

Pero hay un detalle importante: **en este ejemplo, el job utiliza&#x20;**`uses`**&#x20;para llamar a otro workflow y no admite&#x20;**`env`**&#x20;a nivel del job llamador**. Además, las variables `env` del workflow principal no se propagan automáticamente al reutilizable.

Por tanto, para activar el debug de forma fiable en este caso, te recomiendo la siguiente opción.

## **2. Activar el debug desde GitHub (recomendado)**

No necesitas modificar ninguno de los dos archivos YAML.

En tu repositorio, entra en:

**Settings → Secrets and variables → Actions → Variables**

Crea estas dos variables:

| Nombre                 | Valor  | Función                                   |
| ---------------------- | ------ | ----------------------------------------- |
| `ACTIONS_STEP_DEBUG`   | `true` | Activa los logs detallados de los steps   |
| `ACTIONS_RUNNER_DEBUG` | `true` | Activa los logs de diagnóstico del runner |

También puedes crearlas como *Repository secrets*. GitHub admite ambas opciones.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



Después, ejecuta de nuevo `main.yml`. El debug también se aplicará al job del workflow reutilizable.

## **3. Activarlo solo al repetir una ejecución**

También puedes hacerlo sin configurar ninguna variable:

1. Entra en **Actions**.
2. Selecciona una ejecución anterior.
3. Pulsa **Re-run jobs**.
4. Marca **Enable debug logging**.
5. Confirma la ejecución.

Esta opción activa el debug únicamente para esa repetición.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



## **4. Añadir mensajes personalizados de debug**

En `reusable.yml`, puedes añadir un step:

```
- name: Debug parámetros
  run: |
    echo "::debug::Entorno: ${{ inputs.environment }}"
    echo "::debug::Versión: ${{ inputs.version }}"
    echo "::notice::Workflow reutilizable ejecutado"
```

- `::debug::` solo aparece cuando está activado `ACTIONS_STEP_DEBUG`.
- `::notice::` aparece aunque el debug esté desactivado.

No imprimas los valores de los secretos.

**Para tu ejemplo, utilizaría la opción 2**, creando las variables a nivel de repositorio. Así puedes activar y desactivar el debug sin tocar los workflows.