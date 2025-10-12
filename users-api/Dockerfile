# Usar una imagen base de OpenJDK 8
FROM openjdk:8-jdk-alpine

# Instalar Maven
RUN apk add --no-cache maven

# Establecer el directorio de trabajo
WORKDIR /app

# Copiar los archivos del proyecto
COPY . /app

# Construir el JAR (sin mvnw)
RUN mvn clean package -DskipTests

# Variables de entorno
ENV JWT_SECRET=myfancysecret \
    SERVER_PORT=8083

# Exponer puerto
EXPOSE 8083

# Ejecutar la aplicación
CMD ["java", "-jar", "target/users-api-0.0.1-SNAPSHOT.jar"]
