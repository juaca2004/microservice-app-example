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


### **Estructura del manifiesto**

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

4. **Tecnicas de despliegue:**
   Para la creacion de este aplicacion uso 2 tecnicas de depligue. La primera es Rolling Update. Esta es la estrategia de despliegue por defecto y la más común para lograr cero tiempo de inactividad (zero-downtime).
   La uso para la creacion de frontend, Kubernetes despliega los Pods de la nueva versión gradualmente, uno a uno, esperando a que el Pod anterior esté listo antes de terminar con el Pod viejo. Esto asegura que el servicio      siempre esté disponible. La segunda estrategia que uso es Recreate (Recreación). Esta es una estrategia de alto impacto que garantiza que solo una versión de la aplicación esté corriendo en cualquier momento, pero causa      tiempo de inactividad (downtime). La uso en este caso para el deployment auth-api. Kubernetes termina todos los Pods de la versión anterior antes de crear los Pods de la nueva versión.

6. **Network Policies:**
   Se definen **políticas de red de seguridad (NetworkPolicy)** que limitan la comunicación entre servicios para reducir la superficie de ataque:

   * `default-deny-all` para todos los namespaces (bloquea todo tráfico por defecto).
   * Políticas específicas que solo permiten:

     * Tráfico externo → `frontend`.
     * `frontend` → `auth-api` y `todos-api`.
     * `auth-api` → `users-api`.
     * `todos-api` → `redis`.
     * `log-message-processor` → `redis`.

---

### **Seguridad de red**

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

### **Comandos para despliegue**

Asegúrate de estar en el directorio donde se encuentra el archivo `deployment.yaml`.

####  Crear todos los recursos

```bash
kubectl apply -f deployment.yaml
```

####  Verificar los namespaces creados

```bash
kubectl get namespaces
```

####  Comprobar los deployments y pods

```bash
kubectl get deployments --all-namespaces
kubectl get pods --all-namespaces
```

#### Revisar los servicios

```bash
kubectl get svc --all-namespaces
```

#### Verificar las Network Policies

```bash
kubectl get networkpolicy --all-namespaces
```

#### Consultar logs 

```bash
kubectl logs -l app=log-message-processor -n log-ns

```

###  **Grafana y prometheus**

Con la intecion de mayor monitoreo de nuestro cluster, instalamos promtheus y grafana. Gracias estas herramientas nos permitiran monitorear nuestro cluster de manera mas detallar y personalizada, al poder escoger el dashboard que mas nos convenga.

Lo primera que haces es descargar prometheus.
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus -n prometheus --create-namespace
```
Para asegurarnos de que la instalación finalizó con éxito puedes listar los pods.
```bash
kubectl get pods -n prometheus
```
Una vez que tengamos instalado nuestro Prometheus, necesitamos un controlador de Ingress para acceder a la interfaz gráfica. Para habilitar esto en minikube debes ejecutar el siguiente comando:

```bash
minikube addons enable ingress
```
Creamos un servicio para rederigir el trafico en este caso el servicio es un ngnix confiurado en el archivo ingress.yaml

Una vez habilitado debemos aplicar el siguiente manifiesto y añadir una entrada en el /etc/hosts apuntando el dns de nuestro servicio a la ip del Ingress.
<img width="1699" height="1341" alt="image" src="https://github.com/user-attachments/assets/33eda276-786a-4153-b21a-7f852884e84a" />

Luego instalamos grafana siguiendo un procedimiento similar

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana --namespace monitoreo --create-namespace
# 1. Obtener el nombre del secreto
SECRET_NAME=$(kubectl get secret --namespace monitoreo -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

# 2. Decodificar y mostrar la contraseña
kubectl get secret --namespace monitoreo $SECRET_NAME -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Podemos crear un svc para grafana pero perferi hacer un port-forwarding al pod de grafana por practicidad.

Despues de añadir la url de prometheus y importar un dashboard, podemos ver a grafana funcionando.

<img width="1413" height="925" alt="image" src="https://github.com/user-attachments/assets/06223fdd-6f96-4b26-9d01-dd538597ce47" />

<img width="1406" height="634" alt="image" src="https://github.com/user-attachments/assets/b41ae6ed-df5f-491f-a4a2-6b57079d7581" />


EVIDENCIA DE FUNCIONAMIENTO 

<img width="1920" height="933" alt="image" src="https://github.com/user-attachments/assets/2b95c7cf-3ede-4e25-a6d3-2380c9a1671f" />

<img width="1920" height="933" alt="image" src="https://github.com/user-attachments/assets/abe0a070-6d9d-4de3-b8ec-97a508d2a77c" />



