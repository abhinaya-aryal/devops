# Docker

## Container

A docker container is a lightweight standalone, executable package of software that includes everything needed to run an application.

## Docker Install and Setup in Arch Linux

Use your favourite package manager to install docker:

```sh
yay docker
```

Next **enable/start** _docker.service_ or _docker.socket_. Note that **docker.service** starts the service on boot, whereas **docker.socket** starts Docker on first usage which can decrease boot times.

```sh
sudo systemctl enable --now docker.service
```

OR

```sh
sudo systemctl enable --now docker.socket
```

If we want to be able to run the docker CLI command as a **non-root user**, add our user to the **docker** user group, re-login and restart docker.service.

```sh
sudo usermod -aG docker <username>
```

## Common Docker CLI Commands

### docker run

```sh
docker run --interactive --tty --rm ubuntu:22.04
```

Here, `--interactive --tty` or `-it` opens a shell inside a container and attach it. And, `--rm` automatically remove the container when stopped.

After running the command, now we are inside the shell of ubuntu container.

```sh
docker run --interactive --tty --name my-ubuntu-container ubuntu:22.04
```

Here, the `--name` gives the container an identity. It allows us to:

```sh
docker start my-ubuntu-container
docker stop my-ubuntu-container
docker exec -it my-ubuntu-container bash
```

### docker ps

```sh
docker ps
```

This command lists all the **running** container on the device.

```sh
docker ps -a
```

This command lists **all** of the container either stopped, exited or running.

### docker start

```sh
docker start my-ubuntu-container
```

It restarts the container `my-ubuntu-container`.

### docker attach

```sh
docker attach my-ubuntu-container
```

This command attaches the `my-ubuntu-container` to the running shell.

### Create a Custom Image

```sh
docker build --tag my-personal-ubuntu -<<EOF
FROM ubuntu:22.04
RUN apt update && apt install iputils-ping --yes
EOF
```

It creates our **custom** version of **ubuntu** image where **ping** command is already **installed**.

Now, for running a container based on our custom image,

```sh
docker run --it --rm my-personal-ubuntu
```

### Concepts of Unpersisting Data in Docker

```sh
docker run -it --rm ubuntu:22.04
mkdir myData
echo "Hello from container!" > /myData/hello.txt
cat myData/hello.txt

exit
```

Again, we will create a container based on same image as follows:

```sh
docker run -it --rm ubuntu:22.04
cat /myData/hello.txt

exit
```

Here, that file do not exist. It is because of the fact that all the data will be **removed** from the device as soon as container exited.

## Filesystem Mounts

### Bind Mount Vs. Volume Mount

**Volume** is a memory kept by Docker, a **bind mount** is memory kept by us.

Use **Bind Mount** when:

- developing locally
- watching live file changes
- editing code in real time

Use **Volume Mount** when:

- deploying
- persisting databases
- moving across machines

### Bind Mount

```sh
docker run -it --rm --mount type=bind,source="$(pwd)/myData",destination=/myData ubuntu:22.04
```

**Bind mounts** can overwrite container files and file permissions come from the host. Path **must exist and be correct**.

Here, updating the data in host **reflects** the updated data in container too.

### Volume Mount

At first, we will create a volume.

```sh
docker volume create test-volume
```

```sh
docker run -it --rm --mount source=test-volume,destination=/myData/ ubuntu:22.04
echo "Hello from container!" > /myData/hello.txt
cat /myData/hello.txt
exit
```

Now we will recreate a new container mounting the same voulme,

```sh
docker run -it --rm --mount source=test-volume,destination=/myData/ ubuntu:22.04
cat /myData/hello.txt
```

Here, data **persists** across the **container destruction**. Volume is treated separately. Data persists across the boundary of the container.

```sh
/var/lib/docker/volumes
```

Our persisting data lies in the above mentioned directory of the host system.

## Databases

#### i. Use Volumes to persist data:

Generally databases will store its data at one or more known paths. We should identify those and mount volumes to those locations in the containers to ensure data persists beyond the container.

#### ii. Use bind mounts for additional config:

Often databases use configuration files to influence runtime behaviour. We can create these files in our host system, and then use a bind mount to place them in correct location within the container to be read upon startup.

#### iii. Set environment variables:

Many databases use environment variables to influence runtime behaviour. (Eg:- setting the admin password)

### Postgres

```sh
docker run -d --rm \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=foobarbaz \
  -p 5432:5432 \
  postgres:15.1-alpine

# With custom postresql.conf file
docker run -d --rm \
  -v pgdata:/var/lib/postgresql/data \
  -v ${PWD}/postgres.conf:/etc/postgresql/postgresql.conf \
  -e POSTGRES_PASSWORD=foobarbaz \
  -p 5432:5432 \
  postgres:15.1-alpine -c 'config_file=/etc/postgresql/postgresql.conf'
```

### Mongo

```sh
docker run -d --rm \
  -v mongodata:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=foobarbaz \
  -p 27017:27017 \
  mongo:6.0.4

# With custom mongod.conf file
docker run -d --rm \
  -v mongodata:/data/db \
  -v ${PWD}/mongod.conf:/etc/mongod.conf \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=foobarbaz \
  -p 27017:27017 \
  mongo:6.0.4 --config /etc/mongod.conf
```

### Redis

```sh
docker run -d --rm \
  -v redisdata:/data \
  redis:7.0.8-alpine

# With custom redis.conf file
docker run -d --rm \
  -v redisdata:/data \
  -v ${PWD}/redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7.0.8-alpine redis-server /usr/local/etc/redis/redis.conf
```

### MySQL

```sh
docker run -d --rm \
  -v mysqldata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=foobarbaz \
  -p 3306:3306 \
  mysql:8.0.32

# With custom conf.d
docker run -d --rm \
  -v mysqldata:/var/lib/mysql \
  -v ${PWD}/conf.d:/etc/mysql/conf.d \
  -e MYSQL_ROOT_PASSWORD=foobarbaz \
  -p 3306:3306 \
  mysql:8.0.32
```

### Elasticsearch

```sh
docker run -d --rm \
  -v elasticsearchdata:/usr/share/elasticsearch/data
  -e ELASTIC_PASSWORD=foobarbaz \
  -e "discovery.type=single-node" \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:8.6.0
```

### Neo4j

```sh
docker run -d --rm \
    -v=neo4jdata:/data \
    -e NEO4J_AUTH=neo4j/foobarbaz \
    -p 7474:7474 \
    -p 7687:7687 \
    neo4j:5.4.0-community
```

## Interactive Test Environments

### Operating Systems

```sh
# https://hub.docker.com/_/ubuntu
docker run -it --rm ubuntu:22.04

# https://hub.docker.com/_/debian
docker run -it --rm debian:bullseye-slim

# https://hub.docker.com/_/alpine
docker run -it --rm alpine:3.17.1

# https://hub.docker.com/_/busybox
docker run -it --rm busybox:1.36.0 # small image with lots of useful utilities
```

### Programming runtimes

```sh
# https://hub.docker.com/_/python
docker run -it --rm python:3.11.1

# https://hub.docker.com/_/node
docker run -it --rm node:18.13.0

# https://hub.docker.com/_/php
docker run -it --rm php:8.1

# https://hub.docker.com/_/ruby
docker run -it --rm ruby:alpine3.17
```

## CLI Utilities

### Amazon Web Services CLI

```sh
# Bind mount the credentials into the container
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli:2.9.18 s3 ls
```

### Google Cloud Platform CLI

```sh
# Bind mount the credentials into the container
docker run --rm -v ~/.config/gcloud:/root/.config/gcloud gcr.io/google.com/cloudsdktool/google-cloud-cli:415.0.0 gsutil ls
# The container image so big 😭?! 2.8GB
```

## Dockerfile

## Docker Compose file

```sh
# docker-compose.yml
services:
  client-react-vite:
    image: client-react-vite
    build:
      context: ../05-example-web-application/client-react/
      dockerfile: ../../06-building-container-images/client-react/Dockerfile.3
    init: true
    volumes:
      - ./client-react/vite.config.js:/usr/src/app/vite.config.js
    networks:
      - frontend
    ports:
      - 5173:5173
  client-react-nginx:
    labels:
      shipyard.primary-route: true
      shipyard.route: '/'
    image: client-react-nginx
    build:
      context: ../05-example-web-application/client-react/
      dockerfile: ../../06-building-container-images/client-react/Dockerfile.5
    init: true
    networks:
      - frontend
    ports:
      - 80:8080
    restart: unless-stopped
  api-node:
    labels:
      shipyard.route: '/api/node/'
      shipyard.route.rewrite: true
    image: api-node
    build:
      context: ../05-example-web-application/api-node/
      dockerfile: ../../06-building-container-images/api-node/Dockerfile.7
    init: true
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://postgres:foobarbaz@db:5432/postgres
    networks:
      - frontend
      - backend
    ports:
      - 3000:3000
    restart: unless-stopped
  api-golang:
    labels:
      shipyard.route: '/api/golang/'
      shipyard.route.rewrite: true
    image: api-golang
    build:
      context: ../05-example-web-application/api-golang/
      dockerfile: ../../06-building-container-images/api-golang/Dockerfile.6
    init: true
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://postgres:foobarbaz@db:5432/postgres
    networks:
      - frontend
      - backend
    ports:
      - 8080:8080
    restart: unless-stopped
  db:
    image: postgres:15.1-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=foobarbaz
    networks:
      - backend
    ports:
      - 5432:5432
volumes:
  pgdata:
networks:
  frontend:
  backend:
```

## Docker compose for development mode

```sh
# Dockerfile for frontend dev mode
FROM node:19.4-bullseye AS build

# Specify working directory other than /
WORKDIR /usr/src/app

# Copy only files required to install
# dependencies (better layer caching)
COPY package*.json ./

# Use cache mount to speed up install of existing dependencies
RUN --mount=type=cache,target=/usr/src/app/.npm \
  npm set cache /usr/src/app/.npm && \
  npm install

COPY . .

CMD ["npm", "run", "dev"]
```

```sh
# Docker file for both dev and production for backend
# Pin specific version for stability
# Use slim for reduced image size
FROM node:19.6-bullseye-slim AS base

# Specify working directory other than /
WORKDIR /usr/src/app

# Copy only files required to install
# dependencies (better layer caching)
COPY package*.json ./

FROM base as dev

RUN --mount=type=cache,target=/usr/src/app/.npm \
  npm set cache /usr/src/app/.npm && \
  npm install

COPY . .

CMD ["npm", "run", "dev"]

FROM base as production

# Set NODE_ENV
ENV NODE_ENV production

# Install only production dependencies
# Use cache mount to speed up install of existing dependencies
RUN --mount=type=cache,target=/usr/src/app/.npm \
  npm set cache /usr/src/app/.npm && \
  npm ci --only=production

# Use non-root user
# Use --chown on COPY commands to set file permissions
USER node

# Copy the healthcheck script
COPY --chown=node:node ./healthcheck/ .

# Copy remaining source code AFTER installing dependencies.
# Again, copy only the necessary files
COPY --chown=node:node ./src/ .

# Indicate expected port
EXPOSE 3000

CMD [ "node", "index.js" ]
```

```sh
# docker-compose-dev.yml
services:
  client-react-vite:
    image: client-react-vite
    build:
      context: ../05-example-web-application/client-react/
      dockerfile: ../../06-building-container-images/client-react/Dockerfile.3
    init: true
    volumes:
      - type: bind
        source: ../05-example-web-application/client-react/
        target: /usr/src/app/
      - type: volume
        target: /usr/src/app/node_modules
      - type: bind
        source: ../08-running-containers/client-react/vite.config.js
        target: /usr/src/app/vite.config.js
    networks:
      - frontend
    ports:
      - 5173:5173
  client-react-nginx:
    image: client-react-nginx
    build:
      context: ../05-example-web-application/client-react/
      dockerfile: ../../06-building-container-images/client-react/Dockerfile.5
    init: true
    networks:
      - frontend
    ports:
      - 80:8080
    restart: unless-stopped
  api-node:
    image: api-node
    build:
      context: ../05-example-web-application/api-node/
      dockerfile: ../../06-building-container-images/api-node/Dockerfile.9
      target: dev
    init: true
    volumes:
      - type: bind
        source: ../05-example-web-application/api-node/
        target: /usr/src/app/
      - type: volume
        target: /usr/src/app/node_modules
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://postgres:foobarbaz@db:5432/postgres
    networks:
      - frontend
      - backend
    ports:
      - 3000:3000
    restart: unless-stopped
  api-golang:
    image: api-golang
    build:
      context: ../05-example-web-application/api-golang/
      dockerfile: ../../06-building-container-images/api-golang/Dockerfile.8
      target: dev
    init: true
    volumes:
      - type: bind
        source: ../05-example-web-application/api-golang/
        target: /app/
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://postgres:foobarbaz@db:5432/postgres
    networks:
      - frontend
      - backend
    ports:
      - 8080:8080
    restart: unless-stopped
  db:
    image: postgres:15.1-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=foobarbaz
    networks:
      - backend
    ports:
      - 5432:5432
volumes:
  pgdata:
networks:
  frontend:
  backend:
```

## CI/CD to dockerhub

```sh
name: image-ci

on:
  push:
    branches:
      - 'github-action'
    tags:
      - 'v*'

jobs:
  build-tag-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: |
            sidpalas/devops-directive-docker-course-api-node
          tags: |
            type=raw,value=latest
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value={{date 'YYYYMMDD'}}-{{sha}}

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          file: ./06-building-container-images/api-node/Dockerfile.8
          context: ./05-example-web-application/api-node/
          push: true
          tags: ${{ steps.meta.outputs.tags }}

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'sidpalas/devops-directive-docker-course-api-node:latest'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL'
```

## Compose file for docker swarm

```sh
services:
  client-react-nginx:
    image: sidpalas/devops-directive-docker-course-client-react-nginx:5
    deploy:
      mode: replicated
      replicas: 1
      update_config:
        order: start-first
    init: true
    networks:
      - frontend
    ports:
      - 80:8080
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/ping"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
  api-node:
    image: sidpalas/devops-directive-docker-course-api-node:9
    read_only: true
    deploy:
      mode: replicated
      replicas: 1
      update_config:
        order: start-first
    init: true
    environment:
      - DATABASE_URL_FILE=/run/secrets/database-url
    secrets:
      - database-url
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "node", "/usr/src/app/healthcheck.js"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
  api-golang:
    image: sidpalas/devops-directive-docker-course-api-golang:8
    read_only: true
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        order: start-first
    init: true
    environment:
      - DATABASE_URL_FILE=/run/secrets/database-url
    secrets:
      - database-url
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "/healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
  db:
    image: postgres:15.1-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - PGUSER=postgres
      - POSTGRES_PASSWORD_FILE=/run/secrets/postgres-passwd
    secrets:
      - postgres-passwd
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
volumes:
  pgdata:
networks:
  frontend:
  backend:
secrets:
  database-url:
    external: true
  postgres-passwd:
    external: true
```

## Grafana with prometheus

### Directory Structure

Create a neat home for our monitoring stack:

```sh
mkdir -p ~/monitoring/{prometheus, grafana}
cd ~/monitoring
```

### Prometheus Configuration

Create the file `~/monitoring/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### Docker Compose Setup

Create `~/monitoring/docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped
    pid: "host"
    volumes:
      - /:/host:ro,rslave
    command:
      - "--path.rootfs=/host"

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

### Start the Stack

```sh
cd ~/monitoring
docker compose up -d
```

| Service           | URL                                            | Description                           |
| ----------------- | ---------------------------------------------- | ------------------------------------- |
| **Prometheus**    | [http://server-ip:9090](http://server-ip:9090) | Raw metrics and query UI              |
| **Node Exporter** | [http://server-ip:9100](http://server-ip:9100) | System stats                          |
| **Grafana**       | [http://server-ip:3000](http://server-ip:3000) | Dashboards (login: `admin` / `admin`) |

### Configure Grafana

1. Visit -> `http://server-ip:3000`
2. Log in with:
   - Username: `admin`
   - Password: `admin`
3. Add **Data Source** -> Choose **Prometheus**
   - URL: `http://prometheus:9090`
   - Save & test
4. Import a pre-build dashboard (for system stats):
   - Click **Import Dashboard** in dashboard section
   - Use dashboard ID: **1860** ("Node Exporter Full")
   - Select Prometheus as the data source
   - Click **Import**

   # Loki Logs id: 13639
