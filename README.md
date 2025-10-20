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


### 📂 **Estructura del manifiesto**

El archivo `deployment.yaml` contiene los siguientes recursos de Kubernetes:

1. **Namespaces:**
   Se definen espacios de nombres aislados para cada componente:

   * `frontend-ns`
   * `auth-ns`
   * `users-ns`
   * `todos-ns`
   * `redis-ns`
   * `log-ns`

2. **Deployments y Services:**
   Cada microservicio cuenta con su propio `Deployment` y `Service` tipo `ClusterIP`, garantizando comunicación interna estable mediante `Service DNS`.

3. **HPA (Horizontal Pod Autoscaler):**
   Solo el `frontend` cuenta con un `HPA` configurado para escalar entre 2 y 5 réplicas según uso de CPU.

4. **Network Policies:**
   Se definen **políticas de red de seguridad (NetworkPolicy)** que limitan la comunicación entre servicios para reducir la superficie de ataque:

   * `default-deny-all` para todos los namespaces (bloquea todo tráfico por defecto).
   * Políticas específicas que solo permiten:

     * Tráfico externo → `frontend`.
     * `frontend` → `auth-api` y `todos-api`.
     * `auth-api` → `users-api`.
     * `todos-api` → `redis`.
     * `log-message-processor` → `redis`.

---

### 🔒 **Seguridad de red**

El modelo de seguridad se basa en el principio de **"zero trust"**:

* Cada namespace inicia con una política de **denegación total de tráfico**.
* Solo las rutas explícitamente autorizadas en las NetworkPolicies son permitidas.
* Esto garantiza que cada microservicio solo pueda comunicarse con los componentes que realmente necesita.

---

### ⚙️ **Variables de entorno clave**

| Variable                   | Servicio                       | Descripción                         |
| -------------------------- | ------------------------------ | ----------------------------------- |
| `AUTH_API_ADDRESS`         | frontend                       | URL interna del servicio Auth.      |
| `TODOS_API_ADDRESS`        | frontend                       | URL interna del servicio Todos.     |
| `JWT_SECRET`               | auth-api, todos-api, users-api | Llave para firmar tokens JWT.       |
| `REDIS_HOST`, `REDIS_PORT` | todos-api, log-processor       | Dirección del servicio Redis.       |
| `REDIS_CHANNEL`            | todos-api, log-processor       | Canal para intercambio de mensajes. |

---

### 🚀 **Comandos para despliegue**

Asegúrate de estar en el directorio donde se encuentra el archivo `deployment.yaml`.

#### 1️⃣ Crear todos los recursos

```bash
kubectl apply -f deployment.yaml
```

#### 2️⃣ Verificar los namespaces creados

```bash
kubectl get namespaces
```

#### 3️⃣ Comprobar los deployments y pods

```bash
kubectl get deployments --all-namespaces
kubectl get pods --all-namespaces
```

#### 4️⃣ Revisar los servicios

```bash
kubectl get svc --all-namespaces
```

#### 5️⃣ Verificar las Network Policies

```bash
kubectl get networkpolicy --all-namespaces
```

#### 7️⃣ Consultar logs (por ejemplo, del Log Processor)

```bash
kubectl logs -l app=log-message-processor -n log-ns
```

EVIDENCIA DE FUNCIONAMIENTO 

<img width="1920" height="933" alt="image" src="https://github.com/user-attachments/assets/2b95c7cf-3ede-4e25-a6d3-2380c9a1671f" />

<img width="1920" height="933" alt="image" src="https://github.com/user-attachments/assets/abe0a070-6d9d-4de3-b8ec-97a508d2a77c" />

