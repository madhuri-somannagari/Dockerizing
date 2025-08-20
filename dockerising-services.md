## DOCKERISING SERVICES

#### “Developer environment problem”:
•	Services like RabbitMQ, Redis, Celery, Gunicorn are Linux-native.
•	On Windows/Mac, devs struggle to run them reliably.
•	Solution: Dockerize the stack → devs can run the same environment locally
Developers → use Docker Compose to clone full environment (Django, Gunicorn, Celery, Redis, RabbitMQ) on any laptop (Windows/Linux/Mac).

## Steps:

### Create a compose file: ./docker-compose.yml
```compose.yml

version: '3.8'

services:
  #db:
   # image: mysql:8.0
    #environment:
     # MYSQL_DATABASE: ${MYSQL_DATABASE}
      #MYSQL_USER: ${MYSQL_DATABASE_USER_NAME}
      #MYSQL_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      #MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    #volumes:
    #- mysql_data:/var/lib/mysql
    #ports:
    #- "3306:3306"

  redis:
    image: redis:7
    restart: unless-stopped
  
  rabbitmq:
    image: rabbitmq:3-management
    restart: unless-stopped
    ports:
      - "5672:5672"

  web:
    build:
      context: .
      dockerfile: Dockerfile
    command: gunicorn hiringdogbackend.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - ./:/app/
      - ./docker/entrypoint.sh:/entrypoint.sh
      - ./requirements.txt:/app/requirements.txt
    env_file:
      - ./.env
    depends_on:
    #  - db
      - redis
      - rabbitmq
    ports:
      - "8000:8000"
    restart: unless-stopped

  celery:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A hiringdogbackend worker --loglevel=info
    volumes:
      - ./:/app/
      - ./docker/entrypoint.sh:/entrypoint.sh
      - ./requirements.txt:/app/requirements.txt
    env_file:
      - ./.env
    environment:
      - CELERY_BROKER_URL=amqp://guest:guest@rabbitmq:5672//
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    depends_on:
    #  - db
      - redis
      - rabbitmq
      - web
    restart: unless-stopped

  celery-beat:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A hiringdogbackend beat --loglevel=info
    volumes:
      - .:/app/git-source
      - ./docker/entrypoint.sh:/entrypoint.sh
      - ./requirements.txt:/app/requirements.txt
    env_file:
      - ./.env
    environment:
       - CELERY_BROKER_URL=amqp://guest:guest@rabbitmq:5672//
       - CELERY_RESULT_BACKEND=redis://redis:6379/0
    depends_on:
    #  - db
      - redis
      - rabbitmq
      - web
    restart: unless-stopped

#volumes:
 # mysql_data:

```
### Create a docker file: ./Docker
```dockerfile

# Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    default-libmysqlclient-dev \
    pkg-config \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --upgrade pip && pip install --no-cache-dir -r requirements.txt

COPY . .


COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]

```
### Create docker ignore: ./.dockerignore
```.dockerignore

myenv/
venv/
__pycache__/
*.pyc
*.pyo
*.pyd
*.db
*.sqlite3
.env
.git
.gitignore
node_modules/
*.log
*.DS_Store
media/*
logs/*

```
### Create a entrypoint: ./docker/entrypoint.sh
```sh

#!/bin/bash
set -e

cd /app

# Run migrations
python manage.py migrate --noinput
python manage.py runserver 0.0.0.0:8000

exec "$@"

```
### Create a .env 
```.env

CLIENT_ID=XXXXXX.apps.googleusercontent.com
CLIENT_SECRET=GOCSPX-XXXXX
PROJECT_ID=test-hdip-XX
JS_ORIGIN=http://localhost:8000
REDIRECT_URI=http://localhost:8000/api/google-auth/callback

```

Build and run the app:
``` bash
docker-compose up –build
```
Start in background: ` docker-compose up -d --build `
Scale workers (dev): ` docker-compose up -d --scale celery=3 `

Tails logs:
```
docker-compose logs -f web
```
Stop and remove: 
```
docker-compose down 
```
Restart specific service
```
docker-compose restart celery
```
