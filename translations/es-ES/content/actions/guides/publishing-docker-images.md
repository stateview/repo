---
title: Publicar imágenes de Docker
intro: 'Puedes publicar imágenes de Docker en un registro, tale como Docker Hub o {% data variables.product.prodname_registry %}, como parte de tu flujo de trabajo de integración continua (IC).'
product: '{% data reusables.gated-features.actions %}'
redirect_from:
  - /actions/language-and-framework-guides/publishing-docker-images
versions:
  free-pro-team: '*'
  enterprise-server: '>=2.22'
  github-ae: '*'
type: tutorial
topics:
  - Packaging
  - Publishing
  - Docker
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}
{% data reusables.actions.ae-beta %}

### Introducción

Esta guía te muestra cómo crear un flujo de trabajo que realiza una compilación de Docker y posteriormente publica las imágenes de Docker en Docker Hub o {% data variables.product.prodname_registry %}. Con un solo flujo de trabajo, puedes publicar imágenes a un solo registro o a varios de ellos.

{% note %}

**Nota:** Si quieres subir otro registro de terceros de Docker, el ejemplo en la sección "[Publicar imágenes en {% data variables.product.prodname_registry %}](#publishing-images-to-github-packages)" puede servir como plantilla.

{% endnote %}

### Prerrequisitos

Te recomendamos que tengas una comprensión básica de las opciones de configuración de flujo de trabajo y cómo crear un archivo de flujo de trabajo. Para obtener más información, consulta la sección "[Aprende sobre {% data variables.product.prodname_actions %}](/actions/learn-github-actions)".

También puede que encuentres útil el tener un entendimiento básico de lo siguiente:

- "[Secretos cifrados](/actions/reference/encrypted-secrets)"
- "[Autenticación en un flujo de trabajo](/actions/reference/authentication-in-a-workflow)"
- "[Trabajar con el registro de Docker](/packages/working-with-a-github-packages-registry/working-with-the-docker-registry)"

### Acerca de la configuración de imágenes

Esta guía asume que tienes una definición completa de una imagen de Docker almacenada en un repositorio de {% data variables.product.prodname_dotcom %}. Por ejemplo, tu repositorio debe contener un _Dockerfile_, y cualquier otro archivo que se necesite para realizar una compilación de Docker para crear una imagen.

En esta guía, utilizaremos la acción `build-push-action` de Docker para compilar la imagen de Docker y cargarla a uno o más registros de Docker. Para obtener más información, consulta la sección [`build-push-action`](https://github.com/marketplace/actions/build-and-push-docker-images).

{% data reusables.actions.enterprise-marketplace-actions %}

### Publicar imágenes en Docker Hub

{% data reusables.github-actions.release-trigger-workflow %}

En el flujo de trabajo de ejemplo a continuación, utilizamos las acciones `login-action` y `build-push-action` para crear la imagen de Docker y, si la compilación es exitosa, subimos la imagen compilada a Docker Hub.

Para hacer una carga en Docker hub, necesitarás tener una cuenta de Docker Hub y haber creado un repositorio ahí mismo. Para obtener más información, consulta la sección "[Publicar una imagen de contenedor de Docker en Docker Hub](https://docs.docker.com/docker-hub/repos/#pushing-a-docker-container-image-to-docker-hub)" en la documentación de Docker.

Las opciones de `login-action` que se requieren para Docker hub son:

* `username` y `password`: Este es tu nombre de usuario y contraseña de Docker Hub. Te recomendamos almacenar tu nombre de usuario y contraseña de Docker Hub como un secreto para que no se expongan en tu archivo de flujo de trabajo. Para más información, consulta "[Crear y usar secretos cifrados](/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets)."

Las opciones de `build-push-action` que se requieren para Docker Hub son:
* `tags`: La etiqueta de tu nueva imagen en el formato `DOCKER-HUB-NAMESPACE/DOCKER-HUB-REPOSITORY:VERSION`. Puedes configurar una etiqueta sencilla como se muestra a continuación o especificar etiquetas múltiples en una lista.
* `push`: Si se configura como `true`, se subirá la imagen al registro si se compila con éxito.

{% raw %}
```yaml{:copy}
name: Publish Docker image
on:
  release:
    types: [published]
jobs:
  push_to_registry:
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v2
      - name: Log in to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Push to Docker Hub
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: my-docker-hub-namespace/my-docker-hub-repository:latest
```
{% endraw %}

{% data reusables.github-actions.docker-tag-with-ref %}

### Publicar imágenes en {% data variables.product.prodname_registry %}

{% data reusables.github-actions.release-trigger-workflow %}

En el flujo de trabajo de ejemplo a continuación, utilizamos las acciones `login-action` y `build-push-action` de Docker para crear la imagen de Docker y, si la compilación es exitosa, para subir la imagen al {% data variables.product.prodname_registry %}.

Las opciones de `login-action` que se requieren para el {% data variables.product.prodname_registry %} son:
* `registry`: Debe configurarse como `docker.pkg.github.com`.
* `username`: Puedes utilizar el contexto {% raw %}`${{ github.actor }}`{% endraw %} para utilizar automáticamente el nombre de usuario del usuario que desencadenó la ejecución del flujo de trabajo. Para obtener más información, consulta la sección "[Sintaxis de contexto y expresión para GitHub Actions](/actions/reference/context-and-expression-syntax-for-github-actions#github-context)".
* `password`: Puedes utilizar el secreto generado automáticamente `GITHUB_TOKEN` para la contraseña. Para más información, consulta "[Autenticando con el GITHUB_TOKEN](/actions/automating-your-workflow-with-github-actions/authenticating-with-the-github_token)."

Las opciones de `build-push-action` requeridas para {% data variables.product.prodname_registry %} son:
* `tags`: Debe configurarse en el formato `docker.pkg.github.com/OWNER/REPOSITORY/IMAGE_NAME:VERSION`. Por ejemplo, para una imagen que se llame `octo-image` y esté almacenada en {% data variables.product.prodname_dotcom %} en la ruta `http://github.com/octo-org/octo-repo`, la opción `tags` debe configurarse como `docker.pkg.github.com/octo-org/octo-repo/octo-image:latest`. Puedes configurar una etiqueta sencilla como se muestra a continuación o especificar etiquetas múltiples en una lista.
* `push`: Si se configura como `true`, se subirá la imagen al registro si se compila con éxito.

```yaml{:copy}
name: Publish Docker image
on:
  release:
    types: [published]
jobs:
  push_to_registry:
    name: Push Docker image to GitHub Packages
    runs-on: ubuntu-latest{% if currentVersion == "free-pro-team@latest" or currentVersion ver_gt "enterprise-server@3.1" or currentVersion == "github-ae@next" %}
    permissions:
      packages: write
      contents: read{% endif %}
    steps:
      - name: Check out the repo
        uses: actions/checkout@v2
      - name: Log in to GitHub Docker Registry
        uses: docker/login-action@v1
        with:
          registry: {% if currentVersion == "github-ae@latest" %}docker.YOUR-HOSTNAME.com{% else %}docker.pkg.github.com{% endif %}
          username: {% raw %}${{ github.actor }}{% endraw %}
          password: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}
      - name: Build container image
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: |
            {% if currentVersion == "github-ae@latest" %}docker.YOUR-HOSTNAME.com{% else %}docker.pkg.github.com{% endif %}{% raw %}/${{ github.repository }}/octo-image:${{ github.sha }}{% endraw %}
            {% if currentVersion == "github-ae@latest" %}docker.YOUR-HOSTNAME.com{% else %}docker.pkg.github.com{% endif %}{% raw %}/${{ github.repository }}/octo-image:${{ github.ref }}{% endraw %}
```

{% data reusables.github-actions.docker-tag-with-ref %}

### Publicar imágenes en Docker Hub y en {% data variables.product.prodname_registry %}

En un flujo de trabajo sencillo, puedes publicar tu imagen de Docker en registros múltiples si utilizas las acciones `login-action` y `build-push-action` para cada registro.

El siguiente flujo de trabajo de ejemplo utiliza los pasos de las secciones anteriores ("[Publicar imágenes en Docker Hub](#publishing-images-to-docker-hub)" y "[Publicar imágenes en el {% data variables.product.prodname_registry %}](#publishing-images-to-github-packages)") para crear un solo flujo de trabajo que cargue ambos registros.

```yaml{:copy}
name: Publish Docker image
on:
  release:
    types: [published]
jobs:
  push_to_registries:
    name: Push Docker image to multiple registries
    runs-on: ubuntu-latest{% if currentVersion == "free-pro-team@latest" or currentVersion ver_gt "enterprise-server@3.1" or currentVersion == "github-ae@next" %}
    permissions:
      packages: write
      contents: read{% endif %}
    steps:
      - name: Check out the repo
        uses: actions/checkout@v2
      - name: Log in to Docker Hub
        uses: docker/login-action@v1
        with:
          username: {% raw %}${{ secrets.DOCKER_USERNAME }}{% endraw %}
          password: {% raw %}${{ secrets.DOCKER_PASSWORD }}{% endraw %}
      - name: Log in to GitHub Docker Registry
        uses: docker/login-action@v1
        with:
          registry: {% if currentVersion == "github-ae@latest" %}docker.YOUR-HOSTNAME.com{% else %}docker.pkg.github.com{% endif %}
          username: {% raw %}${{ github.actor }}{% endraw %}
          password: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}
      - name: Push to Docker Hub
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: my-docker-hub-namespace/my-docker-hub-repository:{% raw %}${{ github.ref }}{% endraw %}
      - name: Build container image
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: {% if currentVersion == "github-ae@latest" %}docker.YOUR-HOSTNAME.com{% else %}docker.pkg.github.com{% endif %}{% raw %}/${{ github.repository }}/my-image:${{ github.ref }}{% endraw %}
```

El flujo de trabajo anterior revisa el repositorio de {% data variables.product.prodname_dotcom %} y utiliza la acción `login-action` dos veces para ingresar en ambos registros y utiliza la acción `build-push-action` dos veces para compilar y subir la imagen de Docker a Docker Hub y al {% data variables.product.prodname_registry %}. Para ambos pasos, etiqueta la imagen de Docker con la referencia de Git para el evento de flujo de trabajo. Este flujo de trabajo se desencadena cuando se publica un lanzamiento de {% data variables.product.prodname_dotcom %}, así que la referencia para ambos registros será la etiqueta de Git para el lanzamiento.
