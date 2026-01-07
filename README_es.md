# Ejemplo avanzado de Kargo

Este es un repositorio de GitOps de un ejemplo de Kargo que muestra técnicas y funciones avanzadas de Kargo. Este ejemplo creará múltiples aplicaciones Argo CD y etapas de Kargo con una canalización para avanzar en los cambios de Git e imágenes a través de múltiples etapas.

## Características

- Un almacén que supervisa tanto un repositorio de contenedores para nuevas imágenes como cambios manifiestos en Git
- Implementación de la canalización por etapas con pruebas A/B
- Promoción de cambios de Git y etiquetas de imágenes
- Verificación con análisis de un punto final HTTP REST
- Sincronización de aplicaciones Argo CD
- Flujo de control Etapa para coordinar la promoción a múltiples Etapas
- Ramas renderizadas

## Requisitos

- Kargo v1.2.x (para versiones anteriores de Kargo, cambie a la rama release-XY)
- Una instancia de Argo CD
- GitHub y un registro de contenedores (GHCR.io)
- `git` y `docker` instalados

## Instrucciones

1. Bifurca este repositorio y luego clónalo localmente (desde tu bifurcación).

2. Ejecute `personalize.sh` para personalizar los manifiestos para usar su nombre de usuario de GitHub y el destino de CD de Argo:

    ```shell
    ./personalize.sh
    ```

3. `git commit` los cambios personalizados:

    ```shell
    git commit -a -m "personalize manifests"
    git push
    ```

4. Crea un repositorio de imágenes de contenedores de libros de visitas en tu cuenta de GitHub.

    La forma más sencilla de crear un nuevo repositorio de imágenes ghcr.io es volver a etiquetar y enviar una imagen existente con su nombre de usuario de GitHub:

    ```shell
    docker buildx imagetools create \
      ghcr.io/akuity/guestbook:latest \
      -t ghcr.io/<yourgithubusername>/guestbook:v0.0.1
    ```

    Ahora tendrá un repositorio de imágenes de contenedores `guestbook` . Por ejemplo: https://github.com/yourgithubusername/guestbook/pkgs/container/guestbook

5. Cambiar el repositorio de imágenes del contenedor del libro de visitas a público.

    En la interfaz de usuario de GitHub, dirígete al repositorio de contenedores "guestbook", a la configuración del paquete y cambia la visibilidad del paquete a público. Esto permitirá a Kargo supervisar este repositorio en busca de nuevas imágenes, sin necesidad de configurar Kargo con las credenciales del repositorio de imágenes del contenedor.

    ![cambiar la visibilidad del paquete](docs/change-package-visibility.png)

6. Descargue e instale la última CLI de [Kargo Releases](https://github.com/akuity/kargo/releases) y Argo CD:

    ```shell
    ./download-cli.sh /usr/local/bin/kargo
    ```

7. Iniciar sesión en Kargo y Argo CD:

    ```shell
    kargo login https://<kargo-url> --admin
    argocd login <argocd-hostname>
    ```

8. Crear el proyecto y las aplicaciones `guestbook` del CD de Argo

    ```shell
    argocd proj create -f ./argocd/appproj.yaml
    argocd appset create ./argocd/appset.yaml
    ```

9. Crear los recursos de Kargo

    ```shell
    kargo apply -f ./kargo
    ```

10. Agregue las credenciales del repositorio git a Kargo (reemplace `<yourgithubusername>` con su nombre de usuario).

    ```shell
    kargo create credentials github-creds \
      --project kargo-advanced \
      --git \
      --username <yourgithubusername> \
      --repo-url https://github.com/<yourgithubusername>/kargo-advanced.git
    ```

    Como parte del proceso de promoción, Kargo requiere privilegios para confirmar cambios en su repositorio de Git. Asegúrese de que el token proporcionado tenga estos privilegios.

11. ¡Promociona la imagen!

    Ahora tienes una canalización de Kargo que promueve imágenes desde el repositorio de imágenes del contenedor del libro de visitas mediante una canalización de implementación de varias etapas. Visita el proyecto `kargo-advanced` en la interfaz de usuario de Kargo para ver la canalización de implementación.

    ![tubería](docs/pipeline.png)

    Para promocionar, haga clic en el icono de destino a la izquierda de la etapa `dev` , seleccione la carga detectada y haga clic en `Yes` para promocionarla. Una vez promocionada, la carga podrá ser promovida a las etapas posteriores ( `staging` , `prod` ).

## Simulación de una liberación

Para simular un lanzamiento, simplemente vuelva a etiquetar una imagen con una versión semántica más nueva, por ejemplo:

```shell
docker buildx imagetools create \
  ghcr.io/akuity/guestbook:latest \
  -t ghcr.io/<yourgithubusername>/guestbook:v0.0.2
```

Luego actualice el almacén en la interfaz de usuario para detectar la nueva carga.

## Promoviendo cambios manifiestos

Para promover un cambio en el manifiesto, edite el contenido del directorio [`base`](./base) . Por ejemplo, modifique `guestbook-deploy.yaml` con una variable de entorno adicional:

```yaml
        env:
        - name: FOO
          value: bar
```

Kargo promoverá la variable de entorno de la misma manera que con las etiquetas de imagen.
