# DevOpsCourse
Rendu des TP du cours DevOps


# TP1 - Docker
## 1-1 Document your database container essentials: commands and Dockerfile

Pour instancier l'image de la base de données, j'utilise le Dockerfile suivant : 
``` Dockerfile
FROM postgres:14.1-alpine 

# On instancie les variables d'environnement POSTGRES_DB et POSTGRES_USER
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr

#On insère les scripts de création et d'insertion de données dans le dossier docker-entrypoint-initdb.d
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
```
Je build l'image docker avec la commande : 
``` bash
docker build -t nathanglmt/myfirstpostgres .
# -t pour donner un tag
# . pour dire que le Dockerfile est dans le dossier courant
```
Pour lancer un container de la base de données (nathanglmt/myfirstpostgres) j'utilise la commande suivante : 
``` bash
docker run \
-p 8888:5432 \ # expose les ports (5432 du docker, 8888 du host)
--name myfirstpostgres \ #  donne un nom au container
-e POSTGRES_PASSWORD=pwd \ # instancie une variable d'environnement
--network app-network \ # indique le network du container
-v ~/CPE/DevOpsCourse/docker/database/data:/var/lib/postgresql/data \ # choisir un volume  
-d \ # mode detached, le container démarre comme un background process
nathanglmt/myfirstpostgres # nom de l'image qui instancie le container
```

## 1-2 Why do we need a multistage build? And explain each step of this dockerfile
On fait du multistage car il est nécessaire de build le java avec un jdk dans un premier temps avant de lancer le jar.

``` yml
# Build
# On utilise une image de maven
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build

# On définit une variable d'environnement MYAPP_HOME et indique comme environnement de travail
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME

# On copie le pom et les sources dans le container
COPY pom.xml .
COPY src ./src

# On package l'application en .jar, sans lancer les tests unitaires
RUN mvn package -DskipTests


# Run
# On utilise une image d'openjdk
FROM amazoncorretto:17

# On définit une variable d'environnement MYAPP_HOME et indique comme environnement de travail
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME

# On copie le jar généré par le container myapp-build dans notre environnement de travail.
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# On démarre le SpringBoot 
ENTRYPOINT java -jar myapp.jar
```

## 1-3 Document docker-compose most important commands
- **docker-compose up:**
Démarrer les conteneurs définis dans le fichier docker-compose.yml.
 - **docker-compose down:**
Arrêtez et supprimez les conteneurs, les réseaux et les volumes définis dans le fichier docker-compose.yml.
- **docker-compose ps:**
Liste l'état des conteneurs définis dans le fichier docker-compose.yml.

## 1-4 Document your docker-compose file
Voici mon docker-compose :
```yml
version: "3.7"

services:
  backend:
    build:
      context: ./backend/simple-api-student-main # Chemin vers le Dockerfile pour construire l'image
    ports: 
      - "8081:8080" # Mapping de port : port 8081 de l'hôte -> port 8080 du container
    networks:
      - app-network # Liaison au réseau spécifié
    depends_on:
      - database # Dépendance envers le service 'database'
    container_name: backendapistudent # Nom du container

  database:
    build:
      context: ./database # Chemin vers le Dockerfile pour construire l'image
    environment:
      - POSTGRES_PASSWORD=pwd # Définition de la variable d'environnement
    ports: 
      - "8888:5432" # Mapping de port : port 8888 de l'hôte -> port 5432 du container
    networks:
      - app-network # Liaison au réseau spécifié
    volumes:
      - dataDir:/var/lib/postgresql/data # Montage d'un volume pour stocker les données persistantes
    container_name: database # Nom du container

  httpd:
    build:
      context: ./httpd/website # Chemin vers le Dockerfile pour construire l'image
    ports:
      - "5050:80" # Mapping de port : port 5050 de l'hôte -> port 80 du container
    networks:
      - app-network # Liaison au réseau spécifié
    depends_on:
      - backend # Dépendance envers le container 'backend'
    container_name: mypreciouswebsite # Nom du conteneur

  adminer:
    image: adminer # Utilisation de l'image Docker 'adminer' directement
    restart: always # Redémarrage automatique du conteneur en cas d'échec
    networks:
      - app-network # Liaison au réseau spécifié
    ports:
      - 8090:8080 # Mapping de port : port 8090 de l'hôte -> port 8080 du container

networks:
  app-network:
    # Réseau pour que les containers communiquent entre eux

volumes:
  dataDir:
    # Volume pour stocker les données persistantes de la base de données.


```

## 1-5 Document your publication commands and published images in dockerhub
```
docker tag nathanglmt/database nathanglmt/postgres-database:1.0
docker tag nathanglmt/backapistudent nathanglmt/backapistudent:1.0
docker tag nathanglmt/httpd nathanglmt/httpd:1.0
```
Ces commandes enregistent les images (anaisdelcamp/database par exemple) sous un autre nom suivit du numero de la version de l'image (anaisdelcamp/tp1-database:1.0)
```
docker push nathanglmt/postgres-database:1.0
docker push nathanglmt/backapistudent:1.0
docker push nathanglmt/httpd:1.0
```

# TP2 - Github Actions
## 2-1 What are testcontainers
Testcontainers est une bibliothèque Java qui permet d’instancier un conteneur Docker lors des tests, qui permet de fournir des instances légères et jetables de base de données. 

## 2-2 Document your Github Actions configurations
```yml
name: CI devops 2024 # Nom du pipeline

# Lancement du pipeline sur l'évènement push sur les branches main et develop
on:
  push:
    branches: 
      - main
      - develop
jobs:
  test-backend: 
    # Définition d'un job nommé 'test-backend'.
    runs-on: ubuntu-22.04
    # Spécification de l'environnement d'exécution : Ubuntu 22.04.

    steps:
      # Étapes du job :

      - name: Checkout code
        # Étape pour récupérer le code source du dépôt.
        uses: actions/checkout@v2.5.0
        # Utilisation de l'action 'checkout' pour récupérer le code.

      - name: Set up JDK 17
        # Étape pour configurer JDK 17.
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'adopt'
          # Configuration de JDK 17 avec AdoptOpenJDK.

      - name: Build and test with Maven
        # Étape pour construire et tester avec Maven.
        run: mvn clean install --file docker/backend/simple-api-student-main/
        # Commande pour nettoyer le projet, construire et exécuter les tests avec Maven.
        # Paramètre --file pour indiquer le path faire Dockerfile
```

## 2-3 Document your quality gate configuration


# TP3 - Ansible
Contenu de mon setup.yml : 

![image](https://github.com/NathanGlmt/DevOpsCourse/assets/74351197/fff89fc5-f2a4-4adf-b1c0-9938ea61a7fc)

ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"

ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become

![image](https://github.com/NathanGlmt/DevOpsCourse/assets/74351197/03eb6c7e-278d-4753-9f88-1d7e6cba078c)

## 3-2 Document your playbook

playbook.yml
```yml
- hosts: all
  gather_facts: false
  become: true
  
  roles: docker
```

roles/docker/tasks/main.yml
```yml
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

## 3-3 Document your docker_container tasks configuration
inventories/setup.yml
```yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/nathan/CPE/DevOpsCourse/.ssh/id_rsa
   POSTGRES_USR: "usr"
   POSTGRES_DB: "db"
   POSTGRES_PASSWORD: "pwd"
   POSTGRES_URL: "database:5432"
 children:
   prod:
     hosts: centos@nathan.guillemette.takima.cloud
```

roles/docker/tasks/main.yml
```yml
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

roles/create_networks/tasks/main.yml
```yml
---
# tasks file for roles/create_network
- name: Create Docker Network
  docker_network:
    name: my_network
```

roles/launch_app/tasks/main.yml
```yml
---
# tasks file for roles/launch_app
- name: Run App
  community.docker.docker_container:
    name: backendapistudent
    image: nathanglmt/backendapi:1.1
    networks:
      - name: "my_network"
    env:
      POSTGRES_USR: "{{ POSTGRES_USR }}"
      POSTGRES_DB: "{{ POSTGRES_DB }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
      POSTGRES_URL: "{{ POSTGRES_URL }}"
```

roles/launch_database/tasks/main.yml
```yml
---
# tasks file for roles/launch_database
- name: Run Database
  community.docker.docker_container:
    name: database
    image: nathanglmt/postgres-database:1.0
    networks:
      - name: "my_network"
    env:
      POSTGRES_USR: "{{ POSTGRES_USR }}"
      POSTGRES_DB: "{{ POSTGRES_DB }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
    volumes:
      - /data
```

roles/launch_proxy/tasks/main.yml
```yml
---
# tasks file for roles/launch_proxy
- name: Run HTTPD
  community.docker.docker_container:
    name: httpd
    image: nathanglmt/httpd:1.0
    ports:
      - "80:80"
    networks:
      - name: "my_network"
```

playbook.yml
```yml
- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker
    - create_network
    - launch_database
    - launch_app
    - launch_proxy
```

