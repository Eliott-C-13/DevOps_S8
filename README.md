# DevOps_S8
## TP1 
### Question 1-1 :  Document your database container essentials: commands and Dockerfile.
DockerFile :
```
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
```
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
```sudo docker build -t eliott_c/tp1_java_api .```
```sudo docker run -p 8080:8080 --network app-network --name tp1_java eliott_c/tp1_java```

PARTIE HTTP :

Dockerfile :
```
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
```
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
```
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
## TP2

### Question 2-1 : What are testcontainers ?



### Question 2-2 : Document your Github Actions configurations.

### Question 2-3 : Document your quality gate configuration.

