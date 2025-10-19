FROM python:3.8-slim

WORKDIR /app

COPY . .

COPY requirements.txt .

RUN ["pip", "install", "-r", "requirements.txt"]

ENV LOG_PROCESSOR_PORT=8084 \
    REDIS_HOST=localhost \
    REDIS_PORT=6379 \
    REDIS_CHANNEL=log_channel

EXPOSE 8084

CMD ["python", "main.py"]
