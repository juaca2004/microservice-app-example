# Usar imagen base de Go para build
FROM golang:1.20-alpine AS builder

# Establecer directorio de trabajo
WORKDIR /app

# Copiar archivos fuente
COPY . .

# Descargar dependencias
RUN go mod tidy

# Compilar el binario
RUN go build -o auth-api

# Imagen final minimalista
FROM alpine:3.18
WORKDIR /app

# Copiar binario desde la etapa de build
COPY --from=builder /app/auth-api .

# Variables de entorno por defecto (puedes sobreescribirlas en despliegue)
ENV AUTH_API_PORT=8000 \
    USERS_API_ADDRESS=http://users-api:8083 \
    JWT_SECRET=myfancysecret

# Exponer el puerto
EXPOSE 8000

# Comando para ejecutar el servicio
CMD ["./auth-api"]
