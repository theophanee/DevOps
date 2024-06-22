# Build
# Définit l'image de base
FROM maven:3.9.7-eclipse-temurin-21-alpine AS myapp-build
# Variable d'environnement qui pointe vers le réprtoire de l'app
ENV MYAPP_HOME /opt/myapp
# Définit le répertoire de l'app pour exécuter les commandes
WORKDIR $MYAPP_HOME
# Copie les répertoires du build local vers le répertoire de l'image
COPY pom.xml .
COPY src ./src
# Exécute la cmd Maven avec l'option pour ignorer les tests unitaires
RUN mvn package -DskipTests

# Run
# Définit l'image pour l'exécution
FROM openjdk:21-jdk-slim AS myapp-runtime
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# Copie le JAR du build précédent vers le répertoire d'exécution
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# Exécute l'application avec myapp.jar
ENTRYPOINT java -jar myapp.jar