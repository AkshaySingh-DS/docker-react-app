### Docker for a Full-Stack Application with React, Node.js, and PostgreSQL

NOTE: In this application I tried to reduce the size of the images (react and node server) using the multistage build appraoch and using the slim node image.

How does it helps?

- Faster Deployments and Startup Times
  - Lightweight Image, Fewer Layers
- Reduced Image Bloat:
  - All unnecessary build dependencies (e.g., compilers, tools) are discarded before the final image is created, leaving only the essential runtime components.


In the previous example the size of the react and node server images were below:-

```
node-server       v1              4c98c7a6f8aa   4 days ago          1.43GB
react-client      v1              f430d992b266   5 days ago          1.29GB
```

After applying the multistage build it got reduce significantly as per below:-

```
react-client      v3-slim-multi   d278c562fb86   43 minutes ago      279MB
node-server       v5-slim-multi   611c8885a38b   About an hour ago   336MB
```

This repository demonstrates how to set up a React JS, Node JS server with a PostgreSQL database server inside docker containers and connect them all together

To get this project up and running, follow these steps

1. Make sure you have Docker installed in your system. For installation steps, follow the following steps:
    1. For **[Mac](https://docs.docker.com/desktop/install/mac-install/)**
    2. For **[Ubuntu](https://docs.docker.com/engine/install/ubuntu/)**
    3. For **[Windows](https://docs.docker.com/desktop/install/linux-install/)**
2. Clone the repository into your device
3. Open a terminal from the cloned project's directory
4. Strcuture would look like below when you open the project directory


```
├── Dockerfile
├── Readme.md
├── client
│   ├── Dockerfile
│   ├── README.md
│   ├── index.html
│   ├── package-lock.json
│   ├── package.json
│   ├── public
│   │   └── vite.svg
│   ├── src
│   │   ├── App.css
│   │   ├── App.jsx
│   │   ├── assets
│   │   │   └── react.svg
│   │   ├── index.css
│   │   ├── main.jsx
│   │   └── pages
│   │       ├── Home
│   │       │   ├── index.css
│   │       │   └── index.jsx
│   │       └── UserDetail
│   │           ├── index.css
│   │           └── index.jsx
│   └── vite.config.js
├── docker.txt
└── server
    ├── Dockerfile
    ├── README.md
    ├── package-lock.json
    ├── package.json
    ├── prisma
    │   ├── migrations
    │   │   ├── 20240526103908_
    │   │   │   └── migration.sql
    │   │   ├── 20240526104232_
    │   │   │   └── migration.sql
    │   │   ├── 20240526110749_
    │   │   │   └── migration.sql
    │   │   └── migration_lock.toml
    │   ├── schema.prisma
    │   └── seed.js
    └── src
        ├── app.js
        ├── config
        │   └── database.js
        └── constants
            └── httpStatus.js
```

Here is a detailed explanation on Docker what is going on.

#### **1. Introduction**

We have three docker files.
- one for PostgresSQL (We'll build custom image for database)
- one for Node server
- one for react  

NOTE : This application has been tested on local environment so thats not a production level application. It is just a basic way to run this application using docker to understand how docker works with mutiple tech stack together.

This example will work with PostgreSQL as the database, a very minimal Node/Express JS server and React JS as the client side application.

#### **2. Using Docker

When it comes to working with Full Stack Applications, i.e. ones that will involve more than one set of technology to integrate it into one fully fledged system, Docker can be fairly overwhelming to configure from scratch. It is not made any easier by the fact that there are various types of environment dependencies for each particular technology, and it only leads to the risk of errors at a deployment level.

#### **3. Individual Containers**

##### **1. Database**
First of all, the database needs to be set up and running in order for the server to be able to connect to it. The database does not need any Dockerfile in this particular instance, however, it can be done with a Dockerfile too. Lets go through the configurations.

#### Explanation
- ***volumes***: This particular key is for we want to create a container that can **_persist_** data. This means that ordinarily, when a Docker container goes down, so does any updated data on it. Using volumes, we are mapping a particular directory of our local machine with a directory of the container. In this case, that's the directory where postgres is reading the data from for this database.
- ***heathcheck***: when required, certain services will need to check if their state is functional or not. For example, PostgreSQL, has a behavior of turning itself on and off a few instances at launch, before finally being functional. For this reason, healthcheck allows Docker to allow other services to know when it is fully functional.

*`Dockerfile`*
```FROM postgres:latest

ENV POSTGRES_USER=${POSTGRES_USER}
ENV POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
ENV POSTGRES_DB=${POSTGRES_DB}

# Add a health check to ensure PostgreSQL is ready
HEALTHCHECK --interval=5s --timeout=60s --retries=5 --start-period=80s \
  CMD pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB} || exit 1

EXPOSE 5432
```

We'll use below docker command to create a docker image:-

```
docker build -t postgres-custom . 
```
Make sure we should run the above command under the **DOCKER-REACT-NODEJS-POSTGRES** directory where Dockerfile is present

We'll use below docker command to run the above container:-

```
docker run -d \  
  --name database \
  --network my-network \
  -e POSTGRES_USER=john_doe \
  -e POSTGRES_PASSWORD=john.doe \
  -e POSTGRES_DB=docker_test_db \
  -p 5431:5432 \
  -v ./docker_test_db:/var/lib/postgresql/data \
  postgres-custom
```

In above 
-d => detached mode
--network => to run above container on specific custom network **_my-network_**
-e => to pass environment variable at run time (POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB)
-p => to publish the port, 5432 is the port where postgres is acceesible in container and 5431 is the port on host machine
-v => Bind mount as a **_persistrent_volume_**, this would map the local directory where the data is stored with the container directory. Help us to retain the data when container will be removed.

Before running the above container on custom network we first have to create the network using the below command:-

- docker network create my-network

##### **2. Server**

*`Dockerfile`*
```
# Stage1: Build Stage
FROM node:22-slim AS builder
WORKDIR /server
COPY . .
RUN npm install && npx prisma generate

#Stage2: Runtime Stage
FROM node:22-slim
WORKDIR /server

#copy only the necessary files from build stage
COPY --from=builder /server/node_modules ./node_modules
COPY --from=builder /server/prisma/ ./prisma
COPY --from=builder /server/package.json ./package.json
COPY --from=builder /server/src/ ./src
COPY --from=builder /server/package-lock.json ./package-lock.json

ENV DATABASE_URL="postgresql://john_doe:john.doe@database:5432/docker_test_db?schema=public"
ENV PORT=8000

#prisma required below dependecy comes by default with node bigger image
# RUN apt-get update -y && apt-get install -y openssl 
RUN apk update && apk upgrade && apk add --no-cache \
    openssl

EXPOSE 8000
# Command to run when the container starts
CMD ["bash", "-c", "npx prisma migrate reset --force && npm start"]
```
**Explanation**
***FROM*** - tells Docker what image is going to be required to build the container. For this example, its the Node JS (version 18)
***WORKDIR*** - sets the current working directory for subsequent instructions in the Dockerfile. The `server` directory will be created for this container in Docker's environment
***COPY*** - separated by a space, this command tells Docker to copy files/folders ***from local environment to the Docker environment***. The code above is saying that all the contents in the src and prisma folders need to be copied to the `/server/src` & `/srver/prisma` folders in Docker, and package.json to be copied to the `server` directory's root.
***RUN*** - executes commands in the terminal. The commands in the code above will install the necessary node modules, and also generate a prisma client for interacting with the database (it will be needed for seeding the database initially).

We'll use below docker command to create a docker image:-

```
docker build -t node-server . 
```
Make sure we should run the above command under the **server** directory where Dockerfile is present

We'll use below docker command to run the above container:-

```
docker run -d \
  --network my-network \
  --name server \
  -p 7999:8000 \
  node-server
```

In above 
-d => detached mode
--network => to run above container on specific custom network **_my-network_**
-p => to publish the port, 8000 is the port where node server is acceesible in container and 7999 is the port on host machine

##### **3. Client**

*`Dockerfile`*
```
# Stage1: Build Stage
FROM node:22-slim AS builder
ARG VITE_SERVER_URL=http://127.0.0.1:7999
ENV VITE_SERVER_URL=$VITE_SERVER_URL 
WORKDIR /client
COPY . .
RUN npm install && npm run build

#Stage2: Runtime Stage
FROM node:22-slim 
WORKDIR /client

#copy only the necessary files from build stage
COPY --from=builder /client/public/ ./public
COPY --from=builder /client/src/ ./src
COPY --from=builder /client/index.html ./index.html
COPY --from=builder /client/package.json  ./package.json
COPY --from=builder /client/package-lock.json  ./package-lock.json
COPY --from=builder /client/vite.config.js ./vite.config.js
COPY --from=builder /client/node_modules ./node_modules
COPY --from=builder /client/dist ./dist

# Command to run when the container starts
CMD ["bash", "-c", "npm run preview"]
```
**Explanation**
Note: *The commands for `client` are very similar to the already explained above for `server`*
***ARG***: defines a variable that is later passed to the ***ENV*** instruction
***ENV***: Assigns a key value pair into the context of the Docker environment for the container to run. This essentially contains the domain of the API that will be fired from the client later.

We'll use below docker command to create a docker image:-

```
docker build -t react-client .
```
Make sure we must run the above command under the **client** directory where Dockerfile is present

We'll use below docker command to run the above container:-

```
docker run -d \
  --network my-network \
  --name client \
  -p 4172:4173 \
  react-client
```

That's all! That should get the project up and running. To see the output, you can access `http://127.0.0.1:4172` from the browser and you should find a web page with a list of users. This entire system with the client, server & database are running inside of docker and being accessible from your machine.


Application looks something like below:-


<img width="1435" alt="Screenshot 2024-09-14 at 5 55 28 PM" src="https://github.com/user-attachments/assets/a989b505-669e-4ef6-af81-e6500105b8fe">



This tutorial provides a basic understanding of using Docker to manage a full-stack application. Explore the code for further details.