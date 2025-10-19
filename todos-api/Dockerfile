# Usar Node.js como base
FROM node:14-alpine

# Crear directorio de trabajo
WORKDIR /app

# Copiar package.json y package-lock.json primero para instalar dependencias
COPY package*.json ./

# Instalar dependencias
RUN npm install

# Copiar el resto del código
COPY . .

# Exponer el puerto de la API
EXPOSE 8082

# Variables de entorno por defecto (puedes sobreescribirlas al correr el contenedor)
ENV TODO_API_PORT=8082 \
    JWT_SECRET=foo \
    REDIS_HOST=localhost \
    REDIS_PORT=6379 \
    REDIS_CHANNEL=log_channel \
    ZIPKIN_URL=http://127.0.0.1:9411/api/v2/spans

# Comando para iniciar la app
CMD ["node", "server.js"]
