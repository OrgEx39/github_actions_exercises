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