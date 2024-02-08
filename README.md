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

## Backend simple api
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

