# Dockerization of web application

There are 4 services running:

- frontend: Angular - available on port 4200
- backend: Express/node.js - available on port 3000
- db: MySQL - available on port 3306
- phpmyadmin: database manager - available on port 8080

Each service has a separate docker container. The containers are defined in the `docker-compose.yml` file.

Link:

[Applicaltion](https://github.com/zsolt2/foodbear)

## Docker compose:

```yaml
version: '3.1'

services:
  backend:
    build: 
      dockerfile: Dockerfile
      context: ./backend
    environment:
      MYSQL_HOST: '${MYSQL_HOST}'
      MYSQL_PORT: '${MYSQL_PORT}'
      MYSQL_USERNAME: '${MYSQL_USERNAME}'
      MYSQL_PASSWORD: '${MYSQL_PASSWORD}'
      MYSQL_DATABASE: '${MYSQL_DATABASE}'
      PORT: '${BACKEND_PORT}'
      SECRET: '${SECRET}'
    networks:
      - foodbear
    ports:
      - '127.0.0.1:${BACKEND_PORT}:${BACKEND_PORT}'
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build: 
      dockerfile: Dockerfile
      context: .
    environment:
      PORT: '${FRONTEND_PORT}'
      BACKEND_PORT: '${BACKEND_PORT}'
      BACKEND_HOST: '${BACKEND_HOST}'
    networks:
      - foodbear
    ports:
      - '127.0.0.1:${FRONTEND_PORT}:${FRONTEND_PORT}'
    depends_on:
      - backend

  db:
    image: mysql
    command: --default-authentication-plugin=mysql_native_password
    networks: 
       - foodbear
    volumes:
       - ./dbdump:/docker-entrypoint-initdb.d
       - mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: '${MYSQL_PASSWORD}'
      MYSQL_DATABASE: '${MYSQL_DATABASE}'
    ports:
       - 127.0.0.1:${MYSQL_PORT}:${MYSQL_PORT}
    healthcheck:
      test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost"]
      timeout: 10s
      retries: 20

  phpmyadmin:
    image: phpmyadmin
    networks:
       - foodbear
    ports:
      - 127.0.0.1:8080:80
    depends_on:
      - db

networks:
  foodbear:
    driver: bridge
volumes:
  mysql-data:
    driver: local
```

## Build

Whit the `docker-compose build` command, we can build the docker image of the `backend` and `frontend` service. Each directory contains a `Dockerfile` which install the necessary node packages.

## Run

The `docker-compose up` command downloads the necessary docker images, if they are not available locally, and runs the docker containers.

I created a separate network. Each container is part of this network, so they can reference each other based on their service names. 

I also created a docker volume for the database container, so the database is not erased if the container stops. The `./dbdump` directory contains an SQL dump file, this is mounted to the `/docker-entrypoint-initdb.d` directory inside the db. The db imports the SQL dump, and loads the users, tables, constraints, sample data etc... to the database. 

The phpmyadmin and backend container are dependent on the database container. The backend containers runs a *health check* every 10 seconds 20 times. If the health check is successful, then the backend container start running. The phpmyadmin container start running sooner, because it does not have a `service_healthy` condition. In a similar manner the frontend is dependent on the backend. 

The environment variables are defined in the `.env` directory, and passed to the containers.

We can check the config, which `docker-compose config`