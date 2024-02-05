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