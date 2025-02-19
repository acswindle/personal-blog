---
date: "2025-02-18"
draft: false
title: "Containerizing a Go Application"
tags: ["Docker", "Go", "DockerHub"]
categories: ["Backend", "DevOps"]
summary: "Containerizing a Go Application."
weight: 1
cover:
  image: "night-port-cover.jpg"
  relative: true
  hidden: false
  hiddenInSingle: false
---

## Introduction

In this post, we will cover how to containerize a Go application using Docker.
Containerizing out application is useful because it allows us to package
and deploy our application in a way that is portable and easy to distribute.
Any server running docker can pull the image from Docker Hub and run the
application on their own server without having to install Go or any other
dependencies. It also allows us to avoid any issues with compatibility between
our developer setup and the production environment.

## Dockerfile

Let's start by creating a new file in the main directory our our project called
`Dockerfile`. This file will contain the instructions for building our Docker
image.

```Dockerfile
FROM golang:1.24

WORKDIR /app

COPY go.mod ./
COPY go.sum ./
RUN go mod download

COPY main.go ./
COPY database ./database
COPY internal ./internal

RUN go build -o /task-manager

CMD [ "/task-manager" ]
```

First we want to start by specify golang v1.24 as our base image to build upon.
We will set our working directory to `/app` and copy the `go.mod` and `go.sum`.
Next, we want to download all the dependencies using `go mod download`. Then we
want to move over all our source code, which incluedes our `main.go` file and
the `database` and `internal` directories. We compile and build our application
using `go build`. Finally, we will run the application using the `CMD` command
and pass the name of our created binary.

## Docker Compose

Next, we want to create a new file called `docker-compose.yml` in the main
directory of our project. This file will contain the instructions for running
our Docker container.

```docker-compose.yml
version: '3'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - ./db/task-manager.sqlite3:/app/db/task-manager.sqlite3
    environment:
      DATABASE_FILE: ${DATABASE_FILE}
      APP_PORT: ${APP_PORT}
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXPIRE_TIME: ${JWT_EXPIRE_TIME}
```

The `version` field specifies the version of Docker Compose we are using. The
`services` field contains the list of servers we want to run. In our case, we
only have one server for our application. But often we may have multiple servers
in our application, such as a web server and a database server. The `build` field
specifies the directory where our Dockerfile is located, which is the current
in our case. The `ports` field specifies the port we want our application port to
our host machine. If we failed to map a port, we will not be able to communicate
with our application outside of the docker network our application is running in.
The file system within the docker container is ephemeral. The `volumes` field allows
us to map a directory on our host machine to a directory on our docker container.
This will allow us to persist data between runs of our application.
The `environment` field allows to specify the environment variables
in the docker image.
In our case, we want to pass in the environment variables from our `.env` file.
We do this by using the `${VARIABLE_NAME}` notation.

Now that we have our `Dockerfile` and `docker-compose.yml` files,
we can build and run our
application using the following command.

```bash
docker compose up --build
```

This will build our Docker image and run our application in a Docker container. After
it builds the image, it should load up and we can
verify that our application is running.

## Docker Hub

Finally, we can push in our image to Docker Hub. This will allow our production environment
a convenient way to pull our latest application image and run it.

First, we need to head over to [docker hub](https://hub.docker.com/) and create an account
if you don't have one already. Once you have an account, you can create a new repository.
In my case I called my `go-expense-tracker-api`.
Next, we need to add our docker credentials to our local machine. We can do this by running the following command.

```bash
docker login
```

And enter your credentials when prompted.
Next, we need to tag our image. We can do this by running the command

```bash
docker tag <image_name> <username>/<image_name>:<version>
```

In my case it was

```bash
docker tag task-manager-api <username>/go-expense-tracker-api:0.0.1
```

After the command runs, you should be able to see the image in your docker hub repository.

Now, we can specify our `docker-compose.yml` file to pull the image from Docker Hub.

```docker-compose.yml {linenos=table,hl_lines=[5]}
version: '3'

services:
  app:
    image: acswindle09/go-expense-tracker-api:0.0.1
    ports:
      - "8080:8080"
    volumes:
      - ./db/task-manager.sqlite3:/app/db/task-manager.sqlite3
    environment:
      DATABASE_FILE: ${DATABASE_FILE}
      APP_PORT: ${APP_PORT}
      JWT_SECRET: ${JWT_SECRET}
      JWT_EXPIRE_TIME: ${JWT_EXPIRE_TIME}
```

And verify it pulls the image by running `docker compose up`.
(Note: you may need to run `docker compose down` if you have not done so already)

## Conclusion

In this post, we have covered how to containerize a Go application using Docker
, and push our image up to Docker Hub. By containerizing our application, we can
now deploy our application to any server that can run Docker. This will greatly
simply our deployment process and make our application more portable.

In the next post, we will cover how to deploy our application to AWS using EC2.

Until next time! 🍻
