# DevOpsCourse
Rendu des TP du cours DevOps


# TP1 - Docker
## Database

Pour instancier l'image de la base de données, j'utilise le Dockerfile suivant : 
``` Dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
```
Avec la commande : 
```
docker build -t nathanglmt/myfirstpostgres .
```
Pour lancer un container de la base de données (nathanglmt/myfirstpostgres) j'utilise la commande suivante : 
``` bash
docker run \
-p 8888:5432 \
--name myfirstpostgres \
-e POSTGRES_PASSWORD=pwd \
--network app-network \
-v ~/CPE/DevOpsCourse/docker/database/data:/var/lib/postgresql/data \
-d \
nathanglmt/myfirstpostgres
```

## Backend simple api
On fait du multistage car il est nécessaire de build le java avec un jdk dans un premier temps avant de lancer le jar.

```
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
```
version: "3.7"

services:
  backend:
    build:
      context: ./backend/simple-api-student-main
    ports: 
      - "8081:8080"
    networks:
      - app-network
    depends_on:
      - database
    container_name: backendapistudent

  database:
    build:
      context: ./database
    environment:
      - POSTGRES_PASSWORD=pwd
    ports: 
      - "8888:5432"
    networks:
      - app-network
    volumes:
      - dataDir:/var/lib/postgresql/data
    container_name: myfirstpostgres

  httpd:
    build:
      context: ./httpd/website
    ports:
      - "5050:80"
    networks:
      - app-network
    depends_on:
      - backend
    container_name: mypreciouswebsite

  adminer:
    image: adminer
    restart: always
    networks:
      - app-network
    ports:
      - 8090:8080

networks:
  app-network:

volumes:
  dataDir:

```


# TP3 - Ansible
Contenu de mon setup.yml : 
![image](https://github.com/NathanGlmt/DevOpsCourse/assets/74351197/fff89fc5-f2a4-4adf-b1c0-9938ea61a7fc)
