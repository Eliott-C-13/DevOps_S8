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

### Question 1-2 : Why do we need a multistage build? And explain each step of this dockerfile.

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
sudo docker build -t eliott_c/tp1_java_api .
sudo docker run -p 8080:8080 --network app-network --name tp1_java eliott_c/tp1_java

