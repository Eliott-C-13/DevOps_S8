# DevOps_S8
## TP1 : Docker
### Question 1-1 :  Document your database container essentials: commands and Dockerfile.
DockerFile :
```dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

COPY scripts/01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY scripts/02-InsertData.sql /docker-entrypoint-initdb.d
```
- La première ligne permet d'utiliser l'image postgres:14.1-alpine pour le docker
- Ensuite on définit les variables d'environnement utiles comme les paramètres de connexions à la base de données
  (ici on définit les variables d'environnement POSTGRES_USER et POSGRES_PASSWORD et on leur assigne respectivement les valeurs usr et pwd).
- Les instructions copy permettent de copier des éléments de notre machine hôte sur le docker, ici on fait la copie de scripts sql dans le fichier /docker-entrypoint-initdb. les placer dans ce dossier va nous permettre des exécuter au lancement du docker.

Construction du docker :
```
sudo docker build -t eliott_c/tp1_db .
```
Cette commande va permettre de transformer le DockerFile en image docker. 

Lancement du docker :
```
sudo docker run -p 5432:5432 --name tp1_db --network app-network -v /my/own/datadir:/var/lib/postgresql/data eliott_c/tp1_db
```
Une fois le build fait on peut lancer le docker via la commande ```docker run```.
Ici on ajoute plusieurs paramètres à la commande :
- ``` -p ``` pour spécifier le port hôte et celui du docker
- ``` --name ``` pour spécifier un nom au conteneur
- ``` --network ``` pour placer notre conteneur dans le même réseaux qu'un outil adminer pour la visualisation de la base de données
- ``` -v ``` pour monter un volumes sur le dockeur afin de ne pas perdre les données de la base de données si on coupe le docker

Lancement de l'adminer :
```
sudo docker run     -p "8080:8080"     --net=app-network     --name=adminer     -d     adminer
```
Cette commande permet de lancer un docker qui permettra la visualisation de la base de données grâce à l'outil adminer.
On le place dans le même network que notre base de données pour qu'il puisse accéder aux données.

### Question 1-2 : Why do we need a multistage build ? And explain each step of this dockerfile.

Ici on a besoin d'un build multisatge car on veut pouvoir nommer l'étape de build pour pouvoir l'utiliser dans le run.

Dockerfile :
```dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```
Dans ce Dokerfile on commence par une étape de build :
- On récupère une image maeven qu'on va nommer "myapp-build"
- On définit la variable d'environnement "MYAPP_HOME"
- On change le répertoire de travail
- On fait 2 copie de fichier
- On exécute la commande de build

Ensuite on ajoute une étape de run :
- On récupère une image
- On définit la variable d'environnement "MYAPP_HOME"
- On change le répertoire de travail
- On copie le fichier créer par le build

Enfin on définit l'exécutable par défaut du conteneur.

Commandes pour créer l'image et lancer le docker :
- ```sudo docker build -t eliott_c/tp1_java_api .```
- ```sudo docker run -p 8080:8080 --network app-network --name tp1_java eliott_c/tp1_java```

PARTIE HTTP :

Dockerfile :
```dockerfile
FROM httpd:2.4
COPY ./ /usr/local/apache2/htdocs/
```

```
sudo docker build -t eliott_c/tp1_http .
```

```
sudo docker run -dit -p 8090:80 --name tp1_http  eliott_c/tp1_http
```

GET current conf :
```
sudo docker cp  tp1_http:/usr/local/apache2/conf/httpd.conf /tmp/test
```

### Question 1-3 : Why do we need a reverse proxy?

Le reverse proxy va nous servir à n'exposer qu'un seul port sur le réseaux, on a désormais un seul point d'entrée pour notre application.

### Question 1-4 : 

Fichier : docker-compose.yml
```yaml
version: '3.8'

services:
    backend:
        build: 
          context: ../TP1_api
          dockerfile: Dockerfile
        networks: 
          - my-network
        depends_on: 
          - database
        environment:
          - DB_HOSTNAME=database:5432
          - DB=db
          - DB_USER=usr
          - DB_PASSWORD=pwd

    database:
        build:
          context: ../TP1
          dockerfile: Dockerfile
        networks: 
          - my-network
        environment:
          - POSTGRES_DB=db
          - POSTGRES_DB_USER=usr
          - POSTGRES_DB_PASSWORD=pwd

    httpd:
        build:
          context: ../TP1_http
          dockerfile: Dockerfile
        ports: 
          - "8080:80"
        networks: 
          - my-network
        depends_on: 
          - backend

networks:
    my-network: 
```

Ce fichier docker-compose.yml permet de ne plus avoir à lancer les conteneurs un par un en se souciant que chacun est la bonne configuration dans son Dockerfile. Maintenant avec ce fichier on écrit les configurations de tous les conteneurs nécessaires dans ce fichier unique qui une fois rédiger permettra avec une seule commande de build tous les conteneurs et de les run.

run command : ```sudo docker-compose up --build```

### Question 1-5 : Document your publication commands and published images in dockerhub.

Pour chaque projet on doit construire l'image du docker puis lui ajouter un tag avec la commande ```docker tag``` pour lui associer une version et enfin on peut publier l'image sur dockerhub via la commande ```docker push```.

#### DB :
- ```sudo docker build -t eliottc13/tp1_db .```
- ```sudo docker tag eliottc13/tp1_db eliottc13/tp1_db:1.0```
- ```sudo docker push eliottc13/tp1_db:1.0```

#### BACKEND :
- ```sudo docker build -t eliottc13/tp1_java_api .```
- ```sudo docker tag eliottc13/tp1_java_api eliottc13/tp1_java_api:1.0```
- ```sudo docker push eliottc13/tp1_java_api```

#### HTTP :
- ```sudo docker build -t eliottc13/tp1_http .```
- ```sudo docker tag eliottc13/tp1_http eliottc13/tp1_http:1.0```
- ```sudo docker push eliottc13/tp1_http```

#### Modifications dans le docker compose :
```yaml
version: '3.7'

services:
    backend:
        image: eliottc13/tp1_java_api 
        networks: 
          - my-network
        depends_on: 
          - database
        environment:
          - HOSTNAME=database:5432
          - DB=db
          - USER=usr
          - PASSWORD=pwd

    database:
        image: eliottc13/tp1_db
        networks: 
          - my-network

    httpd:
        image: eliottc13/tp1_http
        ports: 
          - "8080:80"
        networks: 
          - my-network
        depends_on: 
          - backend

networks:
    my-network: 
```
Maintenant que nos images docker sont sûr le dépôt de dockerhub on ne va plus spécifier un build pour avoir une image mais directement aller la chercher sur dockerhub en spécifiant le paramètre image.

## TP2 : Github Actions

### Question 2-1 : What are testcontainers ?

Ce sont simplement des bibliothèques Java qui me permettent d'exécuter un certain nombre de conteneurs Docker pendant les tests. Ici, j'utilise le conteneur postgresql pour m'attacher à mon application pendant les tests.

### Question 2-2 : Document your Github Actions configurations.

Fichier : main.yml :
```yaml
name: CI devops 2023
on:
  push:
    branches: 
      - main
      - develop

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'zulu'
  
      - name: Build and test with Maven
        run: |
          cd DevOps/TP1_DevOps/TP1_api/
          #mvn --batch-mode --update-snapshots package
          # on build avec maeven et on fait vérifier les codes par un outil externe : sonar
          # on utilise ici un secret pour le token de sonar 
          mvn -B verify sonar:sonar -Dsonar.projectKey=Eliott-C-13_DevOps_S8 -Dsonar.organization=eliott-c-13 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml


  build-and-push-docker-image:
   needs: test-backend
   runs-on: ubuntu-22.04
  
   steps:
     - name: Login to DockerHub
       run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }} #on utilise des secret pour pas que le username et le token se retrouvent en public 

     - name: Checkout code
       uses: actions/checkout@v2.5.0
  
     - name: Build image and push backend
       uses: docker/build-push-action@v3
       with:
         context: ./DevOps/TP1_DevOps/TP1_api/
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_java_api:latest
         push: ${{ github.ref == 'refs/heads/main' }}
  
     - name: Build image and push database
       uses: docker/build-push-action@v3
       with:
         context: ./DevOps/TP1_DevOps/TP1/
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_db:latest
         push: ${{ github.ref == 'refs/heads/main' }}
  
     - name: Build image and push httpd
       uses: docker/build-push-action@v3
       with:
         context: ./DevOps/TP1_DevOps/TP1_http/
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_http:latest
         push: ${{ github.ref == 'refs/heads/main' }}
```

#### Après le split pipelines :

Fichier : test_backend.yml :
```yaml
name: test_backend
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'zulu'
  
     #finally build your app with the latest command
      - name: Build and test with Maven
        run: |
          cd DevOps/TP1_DevOps/TP1_api/
          #mvn --batch-mode --update-snapshots package
          mvn -B verify sonar:sonar -Dsonar.projectKey=Eliott-C-13_DevOps_S8 -Dsonar.organization=eliott-c-13 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
```

Fichier : publish_push_images_docker.yml
```yaml
name: publish_push_images_docker
on:
  workflow_run:
    workflows: [test_backend]
    types:
      - completed #On attend que le premien pipeline soit finit
    branches:
      - main #On vérifie que ce soit bien la branche main
jobs:
# define job to build and publish docker image
  build-and-push-docker-image:
   # run only when code is compiling and tests are passing
   runs-on: ubuntu-22.04
   if: ${{ github.event.workflow_run.conclusion == 'success' }} #On vérifie que le premier pipeline est terminé avec l'état success
   # steps to perform in job
   steps:
     - name: Login to DockerHub
       run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

     - name: Checkout code
       uses: actions/checkout@v2.5.0
  
     - name: Build image and push backend
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_DevOps/TP1_api/
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_java_api:latest
         push: ${{ github.ref == 'refs/heads/main' }}
  
     - name: Build image and push database
         # DO the same for database
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_DevOps/TP1/
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_db:latest
         push: ${{ github.ref == 'refs/heads/main' }}
  
     - name: Build image and push httpd
       # DO the same for httpd
       uses: docker/build-push-action@v3
       with:
         # relative path to the place where source code with Dockerfile is located
         context: ./DevOps/TP1_DevOps/TP1_http/
         # Note: tags has to be all lower-case
         tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp1_http:latest
         push: ${{ github.ref == 'refs/heads/main' }}
```


### Question 2-3 : Document your quality gate configuration.

#### Création du token dans sonar cloud :
![sonar](https://github.com/Eliott-C-13/DevOps_S8/assets/116546339/8826c7f5-b105-4e8f-b335-490e337bd4e4)

#### Modification du fichier de workflow pour tester le code dans sonar cloud :
```yaml
mvn -B verify sonar:sonar -Dsonar.projectKey=Eliott-C-13_DevOps_S8 -Dsonar.organization=eliott-c-13 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
```

#### Vérification dans le dashboard sonar cloud :
![sonar_cloud_dash](https://github.com/Eliott-C-13/DevOps_S8/assets/116546339/094f129f-f4fa-4413-a4ab-63d27e356b55)

## TP3 : Ansible

### Ping :

- Fichier setup.yml :
  ```yaml
  all:
    vars:
      ansible_user: centos
      ansible_ssh_private_key_file: ./id_rsa
    children:
      prod:
        hosts: centos@eliott.caumon.takima.cloud
  ```
- Command :
  ```ansible all -i inventories/setup.yml -m ping```
- Réponse :
  ```
  centos@eliott.caumon.takima.cloud | SUCCESS => {
       "ansible_facts": {
           "discovered_interpreter_python": "/usr/bin/python"
       },
       "changed": false,
       "ping": "pong"
   }
  ```
### Aficher l'OS de la machine :
- Command : ``` ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*" ```



