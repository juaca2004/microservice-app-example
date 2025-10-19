# Microservice App - PRFT Devops Training

This is the application you are going to use through the whole traninig. This, hopefully, will teach you the fundamentals you need in a real project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io). Although is a TODO application, it is interesting because the microservices that compose it are written in different programming language or frameworks (Go, Python, Vue, Java, and NodeJS). With this design you will experiment with multiple build tools and environments. 

## Components
In each folder you can find a more in-depth explanation of each component:

1. [Users API](/users-api) is a Spring Boot application. Provides user profiles. At the moment, does not provide full CRUD, just getting a single user and all users.
2. [Auth API](/auth-api) is a Go application, and provides authorization functionality. Generates [JWT](https://jwt.io/) tokens to be used with other APIs.
3. [TODOs API](/todos-api) is a NodeJS application, provides CRUD functionality over user's TODO records. Also, it logs "create" and "delete" operations to [Redis](https://redis.io/) queue.
4. [Log Message Processor](/log-message-processor) is a queue processor written in Python. Its purpose is to read messages from a Redis queue and print them to standard output.
5. [Frontend](/frontend) Vue application, provides UI.

## Architecture

Take a look at the components diagram that describes them and their interactions.

![microservice-app-example](/arch-img/Microservices.png)


#Proyecto de Microservicios con Docker y Kubernetes

En este proyecto desarrollé e implementé una arquitectura basada en **microservicios**, utilizando **Docker** para la contenedorización de cada componente y **Kubernetes** para su despliegue.
Cada servicio cumple una función específica dentro del sistema y se comunica con los demás a través de peticiones HTTP o mensajería interna.

---

Estructura del proyecto

Diseñé cinco microservicios principales, cada uno con su propio `Dockerfile` y configuración de despliegue:

| Servicio                  | Lenguaje / Framework      | Puerto | Descripción                                                          |
| ------------------------- | ------------------------- | ------ | -------------------------------------------------------------------- |
| **frontend**              | Node.js (React o similar) | 8080   | Interfaz gráfica del usuario que interactúa con las APIs.            |
| **auth-api**              | Go                        | 8000   | Servicio responsable de la autenticación y generación de tokens JWT. |
| **users-api**             | Java (Spring Boot)        | 8083   | Gestiona los usuarios del sistema.                                   |
| **todos-api**             | Node.js                   | 8082   | Maneja las tareas y se comunica con Redis y Zipkin.                  |
| **log-message-processor** | Python                    | 8084   | Procesa mensajes y logs provenientes de Redis.                       |

---

## 1.Construcción de las imágenes Docker

Para construir las imágenes, me ubiqué en la raíz del proyecto y ejecuté los siguientes comandos.
Cada uno genera una imagen lista para ejecutar su respectivo servicio. Ademas se subid cada uno a dockerHub para su posterior utilizacion en kubectl

```bash
# Construyo la imagen del frontend
docker build -t frontend-service -f ./frontend/Dockerfile .

# Construyo la imagen del Auth API (Go)
docker build -t auth-api-service -f ./auth-api/Dockerfile .

# Construyo la imagen del Users API (Java + Maven)
docker build -t users-api-service -f ./users-api/Dockerfile .

# Construyo la imagen del Todos API (Node.js)
docker build -t todos-api-service -f ./todos-api/Dockerfile .

# Construyo la imagen del Log Processor (Python)
docker build -t log-processor-service -f ./log-message-processor/Dockerfile .
```

---

## 2. Ejecución local de los contenedores

Antes de realizar el despliegue en Kubernetes, probé los contenedores de forma local para asegurarme de que cada servicio funcionara correctamente.

```bash
# Ejecuto el Auth API
docker run -p 8000:8000 auth-api-service

# Ejecuto el Users API
docker run -p 8083:8083 users-api-service

# Ejecuto el Frontend
docker run -p 8080:8080 frontend-service
```

En esta etapa, configuré las variables de entorno necesarias dentro de cada contenedor según las dependencias entre servicios (por ejemplo, la URL del servicio de autenticación en el frontend).

---

## 3. Despliegue en Kubernetes

Para orquestar los servicios y manejar la comunicación entre ellos, preparé el archivo `deployment.yaml`, que define tanto los **Deployments** como los **Services** de Kubernetes.

El archivo `deployment.yaml` define todos los objetos necesarios para desplegar y orquestar mis microservicios dentro del clúster de Kubernetes.

### Namespace
Primero defino un **namespace** llamado `microservices-demo`, que me permite aislar todos los recursos del proyecto dentro del clúster.

### Deployments
Cada microservicio tiene su propio **Deployment** con la configuración necesaria:

- **Frontend:** Utilizo la estrategia `RollingUpdate` con 2 réplicas y un `HorizontalPodAutoscaler (HPA)` que escala de 2 a 5 pods si el uso de CPU supera el 50%.  
- **Auth API:** Empleo la estrategia `Recreate` para garantizar que solo exista una instancia activa por el manejo de tokens.  
- **Todos API, Users API, Log Message Processor y Redis:** Cada uno tiene su propio `Deployment` con una réplica por defecto y las variables de entorno configuradas para su comunicación interna.

### Services
Defino un **Service** tipo `ClusterIP` para cada aplicación, permitiendo la comunicación interna entre pods:

- `frontend` → expone el puerto 80 (redirige al 8080 del contenedor).  
- `auth-api`, `todos-api`, `users-api` → exponen sus respectivos puertos (`8000`, `8082`, `8083`).  
- `redis` → disponible en el puerto `6379`.

Esto asegura que cada servicio pueda ser descubierto dentro del clúster mediante su nombre DNS (`<service-name>.<namespace>.svc.cluster.local`).

### Network Policies
Configuro varias **NetworkPolicies** para reforzar la seguridad del tráfico interno:

- `deny-all-redis`: Niega todo el tráfico entrante a Redis.  
- `allow-api-to-redis`: Solo permite acceso desde `todos-api` y `log-message-processor`.  
- `allow-frontend-to-todos-api`: Restringe el acceso a `todos-api` únicamente al `frontend`.

De esta forma, solo los servicios autorizados pueden comunicarse entre sí, cumpliendo el principio de mínimo privilegio.

###  Horizontal Pod Autoscaler (HPA)
Defino un **HPA** para el `frontend` que ajusta dinámicamente el número de réplicas según el uso de CPU, mejorando la escalabilidad y la disponibilidad del servicio.


Ejecuté el despliegue completo con el siguiente comando:

```bash
kubectl apply -f deployment.yaml
```

Luego verifiqué que los pods estuvieran corriendo:

```bash
kubectl get pods
```

Y comprobé los servicios y sus puertos expuestos:

```bash
kubectl get svc
```

---

## 4. Explicación técnica de cada servicio

### 🔹 Frontend (`frontend/Dockerfile`)

Utilicé una imagen base de `node:8.17.0`.
Configuré el entorno, instalé las dependencias con `npm install`, y realicé la construcción de la aplicación con `npm run build`.
Finalmente, expuse el puerto `8080` y definí las siguientes variables de entorno:

* `PORT=8080`
* `AUTH_API_ADDRESS=http://127.0.0.1:8000`
* `TODOS_API_ADDRESS=http://127.0.0.1:8082`

El servicio se inicia con `npm start`.

---

### Auth API (`auth-api/Dockerfile`)

Para este microservicio usé una imagen base de `golang:1.20-alpine` para la etapa de compilación y luego una imagen final `alpine:3.18` más liviana.
Compilé el binario principal `auth-api` y expuse el puerto `8000`.
Las variables principales que definí fueron:

* `AUTH_API_PORT=8000`
* `USERS_API_ADDRESS=http://users-api:8083`
* `JWT_SECRET=myfancysecret`

Este servicio maneja la autenticación y validación de tokens JWT.

---

### Users API (`users-api/Dockerfile`)

Para este servicio utilicé `openjdk:8-jdk-alpine` e instalé **Maven** para compilar el proyecto Java.
Construí el archivo JAR ejecutando `mvn clean package -DskipTests` y expuse el puerto `8083`.
Definí las siguientes variables:

* `JWT_SECRET=myfancysecret`
* `SERVER_PORT=8083`

El servicio ejecuta el archivo `target/users-api-0.0.1-SNAPSHOT.jar`.

---

### Todos API (`todos-api/Dockerfile`)

Este servicio lo desarrollé en **Node.js**, usando la imagen `node:14-alpine`.
Copié los archivos `package.json`, instalé las dependencias y configuré el entorno para exponer el puerto `8082`.
Las variables por defecto que establecí fueron:

* `TODO_API_PORT=8082`
* `JWT_SECRET=foo`
* `REDIS_HOST=localhost`
* `ZIPKIN_URL=http://127.0.0.1:9411/api/v2/spans`

Finalmente, configuré el comando de inicio `node server.js`.

---

### Log Message Processor (`log-message-processor/Dockerfile`)

Este servicio lo construí con **Python 3.8**, partiendo de la imagen `python:3.8-slim`.
Instalé las dependencias desde `requirements.txt`, expuse el puerto `8084` y configuré las siguientes variables:

* `LOG_PROCESSOR_PORT=8084`
* `REDIS_HOST=localhost`
* `REDIS_CHANNEL=log_channel`

El contenedor ejecuta `python main.py` al iniciarse.

---

## 5. Verificación del despliegue

Para asegurarme de que todos los servicios estaban funcionando correctamente, utilicé los siguientes comandos:

```bash
# Verifico los logs de un pod
kubectl logs <nombre-del-pod>

# Consulto los deployments y servicios activos
kubectl get deployments
kubectl get services

# Escalo un servicio en caso de ser necesario
kubectl scale deployment users-api --replicas=3

# Elimino todos los recursos del despliegue
kubectl delete -f deployment.yaml
```

---

## 6. Arquitectura general

Representé la arquitectura general de la aplicación de la siguiente forma:

```
[ Frontend ] ⇄ [ Auth API ] ⇄ [ Users API ]
      ↓
   [ Todos API ] ⇄ [ Log Processor ]
```

Cada microservicio se ejecuta en su propio contenedor y se comunica con los demás mediante HTTP o Redis.
Gracias a Kubernetes, logré implementar escalabilidad, balanceo de carga y mejor observabilidad de los componentes.

