# Practica: CI/CD de una aplicación
## Entregables

## Descripción de la práctica

Para la práctica se debe implementar un pipeline de CI/CD para una aplicación escrita en el lenguaje de programación que se desee. La aplicación debe ser una aplicación que se pueda ejecutar en un contenedor de Docker. Se tiene que utilizar un servicio de CI/CD que se haya dado en la asignatura (CircleCI, Jenkins o Argo Workflows) aunque se pueden usar otros (Github Actions, Travis CI, etc). La aplicación debe:
1. Estar en un repositorio de GitHub.
2. Utilizar un sistema de gestión de dependencias.
3. Utilizar git flow para la gestión de ramas.
4. Tener un pipeline de CI que:
    1. Construya la aplicación.
    2. Ejecute los tests de la aplicación.
    3. Genere un informe de cobertura de los tests.
    4. Ejecute el linting de la aplicación.
    5. Ejecute un análisis estático de código.
    6. Se ejecute un analisis de vulnerabilidades (usando Snyk o GitGuardian).
    7. Genere un artefacto de la aplicación. (solo en la rama master/main)
    8. Publique el artefacto en un repositorio de artefactos. (solo en la rama master/main)
 5. Se despliegue en en un cluster de Kubernetes usando ArgoCD (o similar) en la rama master/main.

## Descripción de los entregables
### Repositorio en GitHub y gestión de ramas con git flow
Esta configuración esta orientada para ejecutarse en un flujo de trabajo incluyendo las ramas `develop`, `release` y `main`.

En el  workflow se especifica en que rama se debe ejecutar cada tarea mediante filtros.

```
workflows:
  test-deploy:
    jobs:
      - test:
          context: dev
          filters:
            branches:
              only:
                - develop
                - release
                - main
      - create_release:
          context: dev
          requires:
            - test
          filters:
            branches:
              only:
                - release 
      - publish_github:
          context: dev
          requires:
            - test
          filters:
            branches:
              only:
                - main
      - docker_image:
          context: dev
          requires:
            - test
          filters:
            branches:
              only:
                - main
```
### Fichero de configuración CI/CD

Para el desarrollo de esta práctica se ha usado CircleCI en el que se han especificado los diferentes Pipelines:  

#### Test de la aplicación 
 
Se ejecutan en el job `test` usando `pytest` dado que la aplicación esta en `Python`

- **Informe de cobertura**: Para generar y se guardar el informe de cobertura se requiere usar el pip de `coverage` y se ejecuta de la siguiente manera:  
    ```
        # Generar y guardar el archivo de cobertura             
        - run:
            name: Run tests
            command: |
                . venv/bin/activate
                coverage run -m pytest tests
                coverage xml -o coverage.xml
    ```
- **Linting**: Se requiere usar el pip de `flake8` para verificar el codigo y se ejecuta de la siguiente manera:  
    ``` 
        # Ejecutar lint
        - run:
            name: Run lint
            command: |
                . venv/bin/activate
                flake8 . 
    ```   
- **análisis estático de código**: Se utiliza `SonarCloud` para el análisis de calidad, el cual se ejecuta de la siguiente manera:
    ```
    orbs:
    sonarcloud: sonarsource/sonarcloud@2.0.0  
    jobs:
        # Analisis del código estático
        - sonarcloud/scan
    ```
- **Análisis de vulnerabilidades**: Se ha configurado `GitGuardian`.
    ```
        # Escanear vulnerabilidades con gitGuardian
      - run:
          name: Run GitGuardian scan
          command: |
            . venv/bin/activate
            export GITGUARDIAN_API_KEY=$GITGUARDIAN_API_KEY
            ggshield secret scan repo . 
    ```
#### Generar un artefacto de la aplicación. 
El job `publish_github` genera un archivo app.zip. Este archivo se genera solo en la rama main.  
```
      - run:
          name: Generate artifact
          command: |
            mkdir -p artifacts
            zip -r artifacts/app.zip .
      - store_artifacts:
          path: artifacts/app.zip
```
#### Publicar el artefacto en un repositorio de artefactos. 
Se publica como un asset en el release de GitHub, solo en la rama main.  
```
- run:
          name: Publish to GitHub Packages
          command: |
            # Publicar el artefacto en el release de GitHub
            curl -u $GITHUB_USER:$GITHUB_TOKEN \
            -H "Content-Type: application/zip" \
            --data-binary @artifacts/app.zip \
            "https://uploads.github.com/repos/$GITHUB_USER/cicd-practica/releases/${RELEASE_ID}/assets?name=app-${VERSION_TAG}.zip"
```  
#### Construir la aplicación
La imagen Docker se construye y se etiqueta en el job `docker_image`, que depende de la rama `main`.
```
      # Construcción de la imagen
      - run:
          name: Build Docker Image
          command: docker build -t myapp:${VERSION_TAG} .
      # Etiquetado y subida a Docker Hub
      - run:
          name: Tag and Push Image
          command: |
            docker tag myapp:${VERSION_TAG} $DOCKERHUB_USERNAME/myapp:${VERSION_TAG}
            docker push $DOCKERHUB_USERNAME/myapp:${VERSION_TAG}
```
