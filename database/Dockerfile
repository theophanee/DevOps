FROM postgres:14.1-alpine

# configure les valeurs de la BDD
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

# utilise les .sql du dossier pour initialiser la BDD de l'image
COPY sql-scripts/*.sql /docker-entrypoint-initdb.d/
