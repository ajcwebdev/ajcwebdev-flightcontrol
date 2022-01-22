# Example project from [A First Look at Docker](https://dev.to/ajcwebdev/a-first-look-at-docker-3hfg)

[Docker](https://www.docker.com/) is a set of tools that use OS-level virtualization to deliver software in isolated packages called containers. Containers bundle their own software, libraries and configuration files. They communicate with each other through well-defined channels and use fewer resources than virtual machines.

[Flightcontrol](https://flightcontrol.dev/) is a fullstack deployment platform that runs on your AWS account and automatically configures and spins up a Fargate container.

## Outline

* [Node project with an Express server](#node-project-with-an-express-server)
  * [Run server](#run-server)
* [Container image](#container-image)
  * [Dockerfile](#dockerfile)
  * [dockerignore](#dockerignore)
  * [Build project with docker build](#build-project-with-docker-build)
  * [List Docker images with docker images](#list-docker-images-with-docker-images)
* [Run the image](#run-the-image)
  * [Run Docker container with docker run](#run-docker-container-with-docker-run)
  * [List containers with docker ps](#list-containers-with-docker-ps)
  * [Print output of app with docker logs](#print-output-of-app-with-docker-logs)
  * [Call app using curl](#call-app-using-curl)
* [Flightcontrol Config](#flightcontrol-config)
* [Push your project to a GitHub repository](#push-your-project-to-a-github-repository)
  * [Initialize Git](#initialize-git)
  * [Create a new blank repository](#create-a-new-blank-repository)

## Node project with an Express server

We have a boilerplate Node application with Express that returns an HTML fragment.

```javascript
// index.js

const express = require("express")
const app = express()

const PORT = 8080
const HOST = '0.0.0.0'

app.get('/', (req, res) => {
  res.send('<h2>ajcwebdev-docker</h2>')
})

app.listen(PORT, HOST)
console.log(`Running on http://${HOST}:${PORT}`)
```

### Run server

```bash
node index.js
```

## Container image

You'll need to build a Docker image of your app to run this app inside a Docker container using the official Docker image. We will need two files: `Dockerfile` and `.dockerignore`.

### Dockerfile

Docker can build images automatically by reading the instructions from a [`Dockerfile`](https://docs.docker.com/engine/reference/builder/). A `Dockerfile` is a text document that contains all the commands a user could call on the command line to assemble an image. Using `docker build` users can create an automated build that executes several command-line instructions in succession.

The `FROM` instruction initializes a new build stage and sets the Base Image for subsequent instructions. A valid `Dockerfile` must start with `FROM`. The first thing we need to do is define from what image we want to build from. We will use version `14-alpine` of `node` available from [Docker Hub](https://hub.docker.com/_/node) because the universe is chaos and you have to pick something so you might as well pick something with a smaller memory footprint.

```dockerfile
FROM node:14-alpine
```

The `LABEL` instruction is a key-value pair that adds metadata to an image.

```dockerfile
LABEL org.opencontainers.image.source https://github.com/ajcwebdev/ajcwebdev-docker
```

The `WORKDIR` instruction sets the working directory for our application to hold the application code inside the image. 

```dockerfile
WORKDIR /usr/src/app
```

This image comes with Node.js and NPM already installed so the next thing we need to do is to install our app dependencies using the `npm` binary. The `COPY` instruction copies new files or directories from `<src>`. The `COPY` instruction bundles our app's source code inside the Docker image and adds them to the filesystem of the container at the path `<dest>`.

```dockerfile
COPY package*.json ./
```

The `RUN` instruction will execute any commands in a new layer on top of the current image and commit the results. The resulting committed image will be used for the next step in the `Dockerfile`. Rather than copying the entire working directory, we are only copying the `package.json` file. This allows us to take advantage of cached Docker layers.

```dockerfile
RUN npm i
COPY . ./
```

The `EXPOSE` instruction informs Docker that the container listens on the specified network ports at runtime. Our app binds to port `8080` so you'll use the `EXPOSE` instruction to have it mapped by the `docker` daemon.

```dockerfile
EXPOSE 8080
```

Define the command to run the app using `CMD` which defines our runtime. The main purpose of a `CMD` is to provide defaults for an executing container. Here we will use `node index.js` to start our server.

```dockerfile
CMD ["node", "index.js"]
```

Our complete `Dockerfile`.

```dockerfile
FROM node:14-alpine
LABEL org.opencontainers.image.source https://github.com/ajcwebdev/ajcwebdev-docker
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm i
COPY . ./
EXPOSE 8080
CMD [ "node", "index.js" ]
```

### dockerignore

Before the Docker CLI sends the context to the docker daemon, it looks for a file named `.dockerignore` in the root directory of the context. If this file exists, the CLI modifies the context to exclude files and directories that match patterns in it. This helps avoid sending large or sensitive files and directories to the daemon.

```
node_modules
Dockerfile
.dockerignore
.git
.gitignore
npm-debug.log
```

### Build project with docker build

The [`docker build`](https://docs.docker.com/engine/reference/commandline/build/) command builds an image from a Dockerfile and a "context". A build’s context is the set of files located in the specified `PATH` or `URL`. The `URL` parameter can refer to three kinds of resources:
* Git repositories
* Pre-packaged tarball contexts
* Plain text files

Go to the directory with your `Dockerfile` and build the Docker image.

```bash
docker build . -t ajcwebdev-docker
```

The `-t` flag lets you tag your image so it's easier to find later using the `docker images` command.

### List Docker images with docker images

Your image will now be listed by Docker. The [`docker images`](https://docs.docker.com/engine/reference/commandline/images/) command will list all top level images, their repository and tags, and their size.

```bash
docker images
```

## Run the image

Docker runs processes in isolated containers. A container is a process which runs on a host. The host may be local or remote.

### Run Docker container with docker run

When an operator executes [`docker run`](https://docs.docker.com/engine/reference/run/), the container process that runs is isolated in that it has its own file system, its own networking, and its own isolated process tree separate from the host.

```bash
docker run -p 49160:8080 -d ajcwebdev/ajcwebdev-docker
```

`-d` runs the container in detached mode, leaving the container running in the background. The `-p` flag redirects a public port to a private port inside the container.

### List containers with docker ps

To test your app, get the port of your app that Docker mapped:

```bash
docker ps
```

### Print output of app with docker logs

```bash
docker logs <container id>
```

### Call app using curl

```bash
curl -i localhost:49160
```

## Flightcontrol Config

Flightcontrol contains its own infrastructure as code specification that is used in a `flightcontrol.json` file. Documentation is currently in progress.

```json
{
  "environments": [
    {
      "id": "production",
      "name": "Production",
      "region": "us-west-2",
      "source": {
        "branch": "main"
      },
      "services": [
        {
          "id": "my-webapp",
          "name": "my-webapp",
          "type": "fargate",
          "cpu": 0.5,
          "memory": 1024,
          "minInstances": 1,
          "maxInstances": 1,
          "buildCommand": "npm i",
          "startCommand": "node index.js",
          "envVariables": {
            "APP_ENV": "production"
          },
          "port": 8080
        }
      ]
    }
  ]
}
```

## Push your project to a GitHub repository

We can publish this image to the GitHub Container Registry with GitHub Packages. This will require pushing our project to a GitHub repository. Before initializing Git, create a `.gitignore` file for `node_modules` and our environment variables.

```bash
echo 'node_modules\n.DS_Store\n.env' > .gitignore
```

It is a good practice to ignore files containing environment variables to prevent sensitive API keys being committed to a public repo. This is why I have included `.env` even though we don't have a `.env` file in this project right now.

### Initialize Git

```bash
git init
git add .
git commit -m "I can barely contain my excitement"
```

### Create a new blank repository

You can create a blank repository by visiting [repo.new](https://repo.new) or using the [`gh repo create`](https://cli.github.com/manual/gh_repo_create) command with the [GitHub CLI](https://cli.github.com/). Enter the following command to create a new repository, set the remote name from the current directory, and push the project to the newly created repository.

```bash
gh repo create ajcwebdev-docker \
  --public \
  --source=. \
  --remote=upstream \
  --push
```

If you created a repository from the GitHub website instead of the CLI then you will need to set the remote and push the project with the following commands.

```bash
git remote add origin https://github.com/ajcwebdev/ajcwebdev-docker.git
git push -u origin main
```
