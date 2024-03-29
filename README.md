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

Dans ce fichier main.yml on va décrire toutes les étapes nécéssaires à notre pipeline, on va aussi pouvoir conditionner l'exécution de certaines tâches. Par exemple on va vérifier s'il y à un git push sur la branche main ou develop, si c'est le cas on va pouvoir déclencher des actions : les jobs.

Dans ces jobs on spécifie toutes les actions qui doivent-être faites comme installer java, compiler un projet, le faire vérifier par sonar,...

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

Pour plus de lisibilité et faciliter la maintenance il possible de séparer ces actions dans différents pipelines.
Ici on choisit de faire un pipeline pour le backend et un autre pour construire et publier nos images docker. Il possible de conditionner le lancement d'un pipeline.
Par exemple ici on va vouloir lancer notre pipeline docker uniquement si le test backend c'est bien passé :
```yaml  if: ${{ github.event.workflow_run.conclusion == 'success' }} ```

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

Pour pouvoir se connecter à distance sur le serveur ansible utilise une clé ssh.

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
- Réponse :
  ```
  centos@eliott.caumon.takima.cloud | SUCCESS => {
       "ansible_facts": {
           "ansible_distribution": "CentOS",
           "ansible_distribution_file_parsed": true,
           "ansible_distribution_file_path": "/etc/redhat-release",
           "ansible_distribution_file_variety": "RedHat",
           "ansible_distribution_major_version": "7",
           "ansible_distribution_release": "Core",
           "ansible_distribution_version": "7.9",
           "discovered_interpreter_python": "/usr/bin/python"
       },
       "changed": false
   }
  ```
### First Playbook :

Grâce aux playbook il est possible d'automatiser des tâches, ici on lance un ping.

- Fichier playbook.yml :
  ```yaml
  - hosts: all
     gather_facts: false
     become: true
   
     tasks:
      - name: Test connection
        ping:
  ```
- Command : ``` ansible-playbook -i inventories/setup.yml playbook.yml ```
- Réponse :
  ```
   PLAY [all] *************************************************************************************************************************************************************************************************
   
   TASK [Test connection] *************************************************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   PLAY RECAP *************************************************************************************************************************************************************************************************
   centos@eliott.caumon.takima.cloud : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
  ```

### Advanced Playbook :

- Fichier playbook.yml :
  ```yaml
   - hosts: all
     gather_facts: false
     become: true
   
   # Install Docker
     tasks:
   
     - name: Install device-mapper-persistent-data
       yum:
         name: device-mapper-persistent-data
         state: latest
   
     - name: Install lvm2
       yum:
         name: lvm2
         state: latest
   
     - name: add repo docker
       command:
         cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
   
     - name: Install Docker
       yum:
         name: docker-ce
         state: present
   
     - name: Install python3
       yum:
         name: python3
         state: present
   
     - name: Install docker with Python 3
       pip:
         name: docker
         executable: pip3
       vars:
         ansible_python_interpreter: /usr/bin/python3
   
     - name: Make sure Docker is running
       service: name=docker state=started
       tags: docker
  ```
- Command : ``` ansible-playbook -i inventories/setup.yml playbook.yml ```
- Réponse :
  ```
  PLAY [all] *************************************************************************************************************************************************************************************************

   TASK [Install device-mapper-persistent-data] ***************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [Install lvm2] ****************************************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [add repo docker] *************************************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [Install Docker] **************************************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [Install python3] *************************************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [Install docker with Python 3] ************************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [Make sure Docker is running] *************************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   PLAY RECAP *************************************************************************************************************************************************************************************************
   centos@eliott.caumon.takima.cloud : ok=7    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
  ```
- Vérification sur la machine distante :
    command : ``` docker --version ```
    réponse : ``` Docker version 25.0.3, build 4debf41 ```

### Using Roles :

Les rôles ansibles permettent de séparer logiquement nos actions dans des fichiers et de les réutiliser quand cela est nécessaire.

Ici on va créer un rôle docker et y placer toutes les actions qui lui sont liées.
- Command : ``` ansible-galaxy init roles/docker ```
- Réponse :  ``` - Role roles/docker was created successfully  ```

Ensuite il suffit juste d'appeler la tâche docker dans main.yml :

- Fichier : playbook.yml :
  ```yaml
   - hosts: all
     gather_facts: false
     become: true
   
   # Install Docker
     tasks:
   
     - name: Install Docker from role
       include_role:
         name: roles/docker
  ```
- Fichier : roles/docker/tasks/main.yml
  ```yaml
     ---
   # tasks file for roles/docker
     - name: Install device-mapper-persistent-data
       yum:
         name: device-mapper-persistent-data
         state: latest
   
     - name: Install lvm2
       yum:
         name: lvm2
         state: latest
   
     - name: add repo docker
       command:
         cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
   
     - name: Install Docker
       yum:
         name: docker-ce
         state: present
   
     - name: Install python3
       yum:
         name: python3
         state: present
   
     - name: Install docker with Python 3
       pip:
         name: docker
         executable: pip3
       vars:
         ansible_python_interpreter: /usr/bin/python3
   
     - name: Make sure Docker is running
       service: name=docker state=started
       tags: docker
  ```
- command : ``` ansible-playbook -i inventories/setup.yml playbook.yml ```
- réponse :
  ```
  PLAY [all] *************************************************************************************************************************************************************************************************

   TASK [Install Docker from role] ****************************************************************************************************************************************************************************
   
   TASK [roles/docker : Install device-mapper-persistent-data] ************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   TASK [roles/docker : Install lvm2] *************************************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   TASK [roles/docker : add repo docker] **********************************************************************************************************************************************************************
   changed: [centos@eliott.caumon.takima.cloud]
   
   TASK [roles/docker : Install Docker] ***********************************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   TASK [roles/docker : Install python3] **********************************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   TASK [roles/docker : Install docker with Python 3] *********************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   TASK [roles/docker : Make sure Docker is running] **********************************************************************************************************************************************************
   ok: [centos@eliott.caumon.takima.cloud]
   
   PLAY RECAP *************************************************************************************************************************************************************************************************
   centos@eliott.caumon.takima.cloud : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
  ```
  On constate bien que maintenant toutes les tâches d'installation de docker proviennent de : roles/docker.

### Deploy your App :

Fichier playbook.yml :
```yaml
- hosts: all
  gather_facts: false
  become: true

# Install Docker
  tasks:

  - name: Install Docker from role
    include_role:
      name: roles/docker

# Create network
  - name: Create network
    include_role:
      name: roles/network

# Launch database
  - name: Launch database
    include_role:
      name: roles/database

# Launch proxy
  - name: Run HTTPD
    include_role:
      name: roles/proxy

# Launch app
  - name: Launch app
    include_role:
      name: roles/app
```

Fichier roles/docker/tasks/main.yml :
le fichier reste le même

Fichier roles/network/tasks/main.yml :
```yaml
---
# tasks file for roles/network
# Create docker network
- name: Create network
  docker_network:
    name: my-network
    state: present
```

Fichier roles/proxy/tasks/main.yml :
```yaml
- name: Run HTTPD
  docker_container:
    name: httpd
    image: eliottc13/tp1_http
    networks:
      - name: my-network
    ports:
      - "80:80"
```

Fichier roles/app/tasks/main.yml :
```yaml
---
# tasks file for roles/app
# Launch app
- name: backend
  docker_container:
    name: backend
    image: eliottc13/tp1_java_api
    networks:
      - name: my-network
    env:
      HOSTNAME: "{{ HOSTNAME }}"
      DB: "{{ DB }}"
      USER: "{{ USER }}"
      PASSWORD: "{{ PASSWORD }}"
```

Fichier roles/database/tasks/main.yml :
```yaml
---
# tasks file for roles/database
# Launhch postgres database
- name: Launch database
  docker_container:
    name: "{{ database_container_name }}"
    image: eliottc13/tp1_db
    networks:
      - name: my-network
    env:
      POSTGRES_USER: "{{ POSTGRES_USER }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
      POSTGRES_DB: "{{ POSTGRES_DB }}"
```

Ajout des variables d'environnement dans le fichier setup.yml :
```yaml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: ./id_rsa
   database_container_name: database
   HOSTNAME: "{{ database_container_name }}:5432"
   DB: db
   USER: usr
   PASSWORD: pwd
   POSTGRES_DB: db
   POSTGRES_USER: usr
   POSTGRES_PASSWORD: pwd
 children:
   prod:
     hosts: centos@eliott.caumon.takima.cloud
```

### Front :

Changement dans la configuration http :

```
ServerName eliott.caumon@takima.cloud
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass /api/ http://backend:8080/
ProxyPassReverse /api/ http://backend:8080/

ProxyPass / http://front:80/
ProxyPassReverse / http://front:80/

</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```
On ajoute un nouvel endpoint pour l'api et un autre pour le front.


### Continuous deployment :

Pour le déploiement continue on ajoute un nouveau fichier deploy.yml dans le workflow :
```yaml
name: deploy
on:
  #to begin you want to launch this job in main and develop
  workflow_run:
    workflows: [publish_push_images_docker]
    types:
      - completed
    branches:
      - main
jobs:
  deploy: 
    #deploy ansible playbook
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #install ansible
      - name: Install Ansible
        run: sudo apt-get install ansible

      - name: Setting up Vault password
        run: |
          echo "${{ secrets.VAULT_PASSWORD }}" > ~/vault_pass.pem
          chmod 600 ~/vault_pass.pem
      
    #run ansible playbook
      - name: Run Ansible playbook
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -H eliott.caumon.takima.cloud >> ~/.ssh/known_hosts
          echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 400 ~/.ssh/id_rsa
          ansible-playbook -i ./DevOps/TP1_DevOps/TP3/ansible/inventories/setup.yml ./DevOps/TP1_DevOps/TP3/ansible/playbook.yml --vault-password-file=~/vault_pass.pem
```
Ce fichier va permettre de se connecter en ssh au serveur et y installer un os, ansible et lancer le playbook qui contient toutes les étapes de déploiement. Les variables sensibles ont put être cryptée grâce à Vault. 

Commande pour encrypter les variables : ```yaml ansible-vault create vault.yml``` il suffit ensuite d'ajouter un mot de passe et de mettre les informations à crypter dans le fichier.

Fichier vault.yml généré :
 ```yaml
$ANSIBLE_VAULT;1.1;AES256
65306565666334383233656561613630633863343564373833343036656233613761643932626330
3361303234366363623430626430326539333637356365390a373131616638373465373733663530
39663339306265313332303538306439326461656439313630666365353162386265346432386330
6538643133353833630a306163396539646639336264643530663737393137383534663831616662
36613239356430353235373566623537356238613932383561363132653230396261343235323031
63393936323139663830613765323533333966373431343532373163643931663830656635623664
61363665323139656532396133656166643361656338363338353937653664633561313732633734
38323837306262336139306137383565666562613531663931393138613061383965396533313836
65313564323238336335383337333233343963343964386231336362643438626137646531386261
61386364663063313537363133316230366162363336616366653734393139306664373832383633
32396237333761626437323330313033663262353934376232343761616432343965396337333963
35303337366665643138393966393033623637633732393130313134306330346638613638393632
30336533303733383663316439646163626431303763353264633935353239316538613537643764
64316235626638313632643662303066623166636561326333326535306366333338623932656261
363463333439323530663461633433313734
```
Il suffit ensuite d'ajouter les lignes suivantes dans le playbook :
```yaml
vars_files:
    - vault.yml
```


