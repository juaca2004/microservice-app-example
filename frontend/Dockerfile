# Usar Node.js como base
FROM node:8.17.0

# Crear directorio de trabajo
WORKDIR /app

# Copiar package.json y package-lock.json primero para instalar dependencias
COPY package*.json ./

# Instalar dependencias
RUN npm install

# Copiar el resto del código
COPY . .

# Construir la aplicación para producción
RUN npm run build

# Exponer el puerto de la app
EXPOSE 8080

# Variables de entorno por defecto (puedes sobreescribirlas al correr el contenedor)
ENV PORT=8080 \
    AUTH_API_ADDRESS=http://127.0.0.1:8000 \
    TODOS_API_ADDRESS=http://127.0.0.1:8082

# Comando para iniciar la app
CMD ["npm", "start"]
