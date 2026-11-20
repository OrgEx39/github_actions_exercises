Una **Artifact Attestation** (atestación de artefacto) es una especie de **certificado digital de procedencia que GitHub Actions genera para demostrar de dónde viene un archivo compilado y que no ha sido modificado**.

Imagina que tienes un proyecto Java y GitHub Actions genera este archivo:

```
mi-aplicacion-1.0.0.jar
```

¿Cómo puedes saber que ese JAR se ha generado realmente desde tu repositorio, utilizando el código correcto y el workflow autorizado, y que nadie lo ha sustituido por otro?

Ahí entran las **Artifact Attestations**.

## **1. ¿Qué información contiene una attestation?**

Una attestation permite verificar información como:

| Información    | Ejemplo                | ¿Para qué sirve?                           |
| -------------- | ---------------------- | ------------------------------------------ |
| Repositorio    | `my-org/mi-aplicacion` | Identificar el origen del código           |
| Organización   | `my-org`               | Identificar al propietario                 |
| Commit SHA     | `7f4a9d3...`           | Identificar el código concreto utilizado   |
| Workflow       | `build.yml`            | Saber qué automatización generó el archivo |
| Evento         | `push`                 | Saber qué desencadenó la compilación       |
| Workflow run   | `123456789`            | Localizar la ejecución que lo generó       |
| Identidad OIDC | Identidad del workflow | Verificar criptográficamente el origen     |
| Digest SHA-256 | Hash del JAR           | Comprobar la integridad del archivo        |

GitHub genera estas declaraciones utilizando Sigstore y mecanismos de firma criptográfica.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs+1



## **2. ¿Cómo funciona?**

**Repositorio GitHub**

Commit `7f4a9d3...`

**GitHub Actions ejecuta build.yml**

Compila el código con Maven

**mi-aplicacion-1.0.0.jar**

**Artifact Attestation**

Identidad firmada + procedencia + hash del archivo

**Otro usuario verifica el JAR**

Comprueba la firma, el origen y la integridad

La gran ventaja es que **no tienes que confiar únicamente en lo que alguien te dice sobre el archivo**. Puedes comprobar criptográficamente la información asociada a su compilación.

## **3. ¿Qué papel tiene OIDC?**

Esta es la parte más técnica de la explicación.

**OIDC (OpenID Connect) permite que GitHub acredite criptográficamente la identidad del workflow que está ejecutándose.**

Imagina que alguien genera un JAR en su ordenador y escribe un archivo diciendo:

```
repository: my-org/mi-aplicacion
workflow: build.yml
commit: 7f4a9d3
```

Cualquiera podría escribir eso. Por sí solo, **no demuestra nada**.

Con OIDC y Sigstore, la situación cambia:

1. GitHub emite una identidad OIDC asociada a la ejecución del workflow.
2. El sistema de firma utiliza esa identidad para emitir un certificado de corta duración.
3. Se firma criptográficamente una declaración que identifica el artefacto y su procedencia.
4. Otro usuario puede verificar la firma y comprobar que la identidad corresponde al workflow esperado.

Lo importante es que **no basta con modificar el texto de la attestation para falsificar el origen**: la verificación criptográfica detectaría la manipulación.

El token OIDC contiene información de identidad; no se publican las credenciales secretas del workflow.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



## **4. ¿Cómo generas una attestation desde GitHub Actions?**

Por ejemplo, para un proyecto Maven que genera un JAR:

```
name: Build con attestation

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write
  attestations: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Descargar código
        uses: actions/checkout@v4

      - name: Configurar Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Compilar JAR
        run: mvn -B package

      - name: Generar attestation
        uses: actions/attest@v4
        with:
          subject-path: target/*.jar

      - name: Guardar JAR
        uses: actions/upload-artifact@v4
        with:
          name: aplicacion
          path: target/*.jar
```

Fíjate en los permisos:

| Permiso               | Función                              |
| --------------------- | ------------------------------------ |
| `contents: read`      | Permite leer el repositorio.         |
| `id-token: write`     | Permite solicitar la identidad OIDC. |
| `attestations: write` | Permite generar la attestation.      |

Y en esta acción:

```
- uses: actions/attest@v4
  with:
    subject-path: target/*.jar
```

Genera una attestation sobre el JAR indicando su procedencia e integridad. Esto es distinto de `actions/upload-artifact`, que almacena el archivo para su posterior descarga.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



## **5. ¿Cómo compruebas que el JAR es auténtico?**

Supongamos que descargas `mi-aplicacion-1.0.0.jar` de una publicación.

Puedes utilizar GitHub CLI:

```
gh attestation verify mi-aplicacion-1.0.0.jar \
  --repo my-org/mi-aplicacion
```

GitHub CLI verifica la información criptográfica asociada al archivo y comprueba la procedencia declarada.

**Ejemplo de interpretación del resultado**

**Verificación satisfactoria**

El hash del archivo coincide con el sujeto firmado y la identidad criptográfica se verifica.

**Verificación fallida**

Por ejemplo, el archivo ha sido modificado o no existe una attestation válida para él.

También puedes establecer comprobaciones adicionales para exigir un workflow firmante determinado.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



## **6. ¿Significa que el archivo es seguro?**

**No necesariamente.** Esta distinción es fundamental.

Una attestation permite verificar que el artefacto procede del proceso de compilación declarado y que su contenido coincide con el identificado criptográficamente.

Pero no garantiza que:

- El código fuente esté libre de vulnerabilidades.
- El workflow de compilación no contenga errores.
- Las dependencias utilizadas sean seguras.
- El artefacto no incluya funcionalidades maliciosas ya presentes en el código original.

Por ejemplo, si el código fuente contiene una vulnerabilidad, el JAR compilado puede tener una attestation perfectamente válida y seguir siendo vulnerable.

GitHub también advierte que generar attestations sin verificarlas no proporciona por sí solo el beneficio de seguridad esperado.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



## **7. ¿Puedo practicarlo en un repositorio privado?**

Hay una limitación importante: según la documentación actual de GitHub, las artifact attestations están disponibles en repositorios públicos con GitHub Free, Pro y Team, pero para utilizarlas en repositorios privados o internos se requiere **GitHub Enterprise Cloud**.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs



Esto es relevante si estás utilizando un repositorio privado de prácticas para GH-200.

## **Resumen para el examen GH-200**

| Concepto                 | ¿Qué demuestra o proporciona?                                               |
| ------------------------ | --------------------------------------------------------------------------- |
| **Provenance**           | De dónde procede el artefacto y cómo se generó.                             |
| **Integrity**            | Que el contenido verificado coincide con el identificado en la attestation. |
| **Workflow association** | Qué ejecución de GitHub Actions produjo el artefacto.                       |
| **OIDC**                 | Identidad verificable del workflow que genera la attestation.               |
| **Sigstore**             | Infraestructura criptográfica para firmar y verificar attestations.         |

**La idea que debes memorizar:**

**Artifact = el archivo que has generado.**

**Artifact Attestation = la prueba criptográfica de su procedencia e integridad.**

No almacena tus secretos ni información de facturación, y no sustituye un análisis de seguridad del código.

Documentación oficial: [GitHub — Artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations?utm_source=chatgpt.com).