|     Propiedad     | Workflow | Job | Step |
|:-----------------:|:--------:|:---:|:----:|
| runs-on           | ❌       | ✅  | ❌   |
| env               | ✅       | ✅  | ✅   |
| permissions       | ✅       | ✅  | ❌   |
| concurrency       | ✅       | ✅  | ❌   |
| defaults          | ✅       | ✅  | ❌   |
| if                | ❌       | ✅  | ✅   |
| continue-on-error | ❌       | ✅  | ✅   |
| timeout-minutes   | ❌       | ✅  | ✅   |
| strategy          | ❌       | ✅  | ❌   |
| needs             | ❌       | ✅  | ❌   |
| environment       | ❌       | ✅  | ❌   |
| container         | ❌       | ✅  | ❌   |
| services          | ❌       | ✅  | ❌   |
| run               | ❌       | ❌  | ✅   |
| uses              | ❌       | ✅¹ | ✅   |

¹ A nivel de job, uses sirve para llamar a un reusable workflow. En ese caso, no se declara runs-on en el job que realiza la llamada, sino en los jobs del workflow reutilizable.

|            Declaración            | Step actual | Steps posteriores del mismo job |         Otros jobs        |
|:---------------------------------:|:-----------:|:-------------------------------:|:-------------------------:|
| env: a nivel de workflow          | ✅          | ✅                              | ✅                        |
| env: a nivel de job               | ✅          | ✅                              | ❌                        |
| env: a nivel de step              | ✅          | ❌                              | ❌                        |
| echo "result=42" >> "$GITHUB_ENV" | ❌          | ✅                              | ❌ (utilizar job outputs) |
| export result=42 dentro de run    | ✅          | ❌                              | ❌                        |

# Niveles para configurar retención de artifactos

|        Nivel        |                Configuración                |
|:-------------------:|:-------------------------------------------:|
| Enterprise          | Política global de retención.               |
| Organization        | Política de retención de la organización.   |
| Repository          | Configuración de retención del repositorio. |
| Individual artifact | Parámetro retention-days.                   |
