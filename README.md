<h1>Database</h1>
<h3>Basics</h3>

``docker build -t my-postgres-image . `` pour build l'image avec comme nom my-postgres-image  
``docker run -d --name my-postgres-container --network app-network -p 5432:5432 -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd my-postgres-image`` pour démarrer le conteneur de l'image    

Je vérifie que le conteneur est en cours d'exécution avec ``docker ps`` : 
``CONTAINER ID   IMAGE               COMMAND                  CREATED              STATUS              PORTS                    NAMES 48f014228c50   my-postgres-image   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:5432->5432/tcp   my-postgres-container ``.  

J'ai ensuite run l'admirer avec ``docker run -d --name adminer-container --network app-network -p 8080:8080 adminer``.  
Puis sur ma page localhost, la page SQL s'affiche, et je renseigne les données :
- Système : PostgreSQL
- Serveur : my-postgres-container
- Utilisateur : usr
- Mot de passe : pwd
- Base de données : db
Et j'ai donc pu accéder à ma base de donnée.  

<h3>Init database</h3>
J'ai mis les 2 scripts SQL dans le répertoire `/sql-inits/` et j'ai rajouté dans le Dockerfile cette ligne ``COPY sql-scripts/*.sql /docker-entrypoint-initdb.d/``.  

Je peux ensuite faire ``ls /docker-entrypoint-initdb.d/`` pour vérifier que les scripts sont bien chargés.  

Quand je charge ma page localhost j'ai bien mes 2 tables de créer. Je teste aussi avec `psql` :
```
db=# \dt
          List of relations
 Schema |    Name     | Type  | Owner
--------+-------------+-------+-------
 public | departments | table | usr
 public | students    | table | usr
(2 rows)

db=# SELECT * FROM students;
 id | department_id | first_name | last_name 
----+---------------+------------+-----------
  1 |             1 | Eli        | Copter
  2 |             2 | Emma       | Carena
  3 |             2 | Jack       | Uzzi
  4 |             3 | Aude       | Javel
(4 rows)

db=# SELECT * FROM departments;
 id | name
----+------
  1 | IRC
  2 | ETI
  3 | CGP
(3 rows)

```

<h3>Persist data</h3>

Je crée d'abord mon volume `docker volume create postgres-data`.  
Je supprime l'ancien adminer :
- ``docker stop my-postgres-container adminer-container`` 
- ``docker rm my-postgres-container adminer-container``

Puis je démarre un nouveau contener et son adminer : 
- ``docker run -d --name my-postgres-container --network app-network -p 5432:5432 -v postgres-data:/var/lib/postgresql/data -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd my-postgres-image``
- ``docker run -d --name adminer-container --network app-network -p 8090:8080 adminer``
J'utilise bien `-v /my/own/datadir:/var/lib/postgresql/data`.

Sur ma page localhost, j'ai bien mes tables toujours présentes. 

Quand je recréer mon container j'ai toujours mes données.

<h1>Backend API</h1>
<h3>Basics</h3>

J'ai copier-coller le Main.java et je l'ai compilé, j'ai donc mon `Main.class`.  

J'ai donc rajouté dans le Dockerfile 
```java
FROM eclipse-temurin:21-jre-alpine

# Add the compiled java (aka bytecode, aka .class)
COPY Main.class /app/
WORKDIR /app

# Run the Java with: “java Main” command.
CMD ["java", "Main"]
```

Puis je build `docker build -t java-hello-world .` et j'exécute avec `docker run --rm java-hello-world`.  
J'ai bien sur ma console 
``docker run --rm java-hello-world
Hello World!```  

<h3>Multistage build</h3>
<h4>Backend simple api</h4>

J'ai installer Spring Boot avec java21 car je n'avais pas le 17.  
J'ai donc dû modifier le Dockerfile avec :
`FROM maven:3.9.7-eclipse-temurin-21-alpine AS myapp-build` et `FROM openjdk:21-jdk-slim AS myapp-runtime`.  
J'ai ensuite placer le Dockerfile au même niveau que le pom.xml pour lancer le build `docker build -t my-springboot-app .` et j'ai lancé avec `docker run -p 8080:8080 my-springboot-app`.  

<h4>Backend API</h4>

J'ai donc ajusté les valeurs de application.yml comme ceci :
```    
url: jdbc:postgresql://localhost:5432/db
username: usr
password: pwd
driver-class-name: org.postgresql.Driver
```

Pour correspondre à nos valeurs données à la database.  

docker build . -t simple-api
après création du network postgresql :
docker run --name my-springboot-container --network my-network -p 8080:8080 simple-api

et j'ai bien 
``
0	
id	1
firstname	"Eli"
lastname	"Copter"
department	
id	1
name	"IRC"
``

<h1>Http server</h1>
<h3>Choose an appropriate base image.</h3>

``
# Utiliser l'image de base Apache
FROM httpd:2.4

# Copier le fichier de configuration httpd.conf corrigé dans l'image
COPY httpd.conf /usr/local/apache2/conf/httpd.conf

# Copier la page index.html dans le répertoire de contenu web
COPY index.html /usr/local/apache2/htdocs/
``

Je build mon docker avec `docker build -t my-http-server .` puis je run `docker run -d -p 8080:80 --name my-http-container my-http-server`. 
Puis j'ai tester stats, logs et inspect :
- stats : pas grand chose, voir rien du tout
- logs `AH00534: httpd: Configuration error: No MPM loaded.`
- inspect : me donne pleins d'informations comme Id, Path, State etc

<h3>Configuration</h3>

Je télécharge le *httpd.conf* de Docker Desktop et je le place dans mon dossier httpd.

<h3>Reverse proxy</h3>
``
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://backend:8080/
ProxyPassReverse / http://backend:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
``
<h3>Link application</h3>

Je renseigne toutes les informations comme databse, backend et httpd.
``
services:
  database:
    build:
      context: ./database
    environment:
      POSTGRES_DB: db
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
    networks:
      - my-network

  backend:
    build:
      context: ./simple-api-student
    networks:
      - my-network
    depends_on:
      - database
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://database:5432/db
      SPRING_DATASOURCE_USERNAME: usr
      SPRING_DATASOURCE_PASSWORD: pwd

  httpd:
    build:
      context: ./httpd
    ports:
      - "8080:80"
    networks:
      - my-network
    depends_on:
      - backend

networks:
  my-network:
``

Puis je lance mon `docker compose up --build` qui va donc lancer tout mes DockerFile et je peux enfin voir mon application sur *http://localhost:8080/*.  

<h3>Publish</h3>

J'enchaîne les commandes suivantes : 
> \> docker login  
Authenticating with existing credentials...  
Login Succeeded  
> \> docker tag tp-httpd tcaraux/my-database:1.0   
> \> docker push tcaraux/my-database:1.0  
> \> docker tag tp-httpd tcaraux/httpd:2.4   
> \> docker push tcaraux/httpd:2.4  
> \> docker tag tp-httpd tcaraux/backend:lastest
> \> docker push tcaraux/backend:lastest  

Ainsi, j'obtiens sur Docker Hub :  
![Texte alternatif](images/publish.png)  


