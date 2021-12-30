# 🐳 Docker

## Overview
- [Docker Layer Caching for Dependencies](#docker-layer-caching-for-dependencies)
    - [How Docker Builds Images](#how-docker-builds-images)
    - [Layer Caching Modules](#layer-caching-modules)

## Docker Layer Caching for Dependencies
The dependencies for an application can take quite some time to install. This is especially true for `NodeJS`.

Because of this, Docker images can take a while to build, always reinstalling dependencies. This can hinder development and deployment. To speed up the build process, we can cache the dependencies using Docker layer caching.

### How Docker Builds Images
Docker builds images in layers. Each layer is a tarball of the contents of the directory. The first layer is the base image, and the last layer is the final image. Each layer after the first is built on top of the previous layer.

### Layer Caching Modules
Docker optimises the build process for most users already by caching the base image that an image is built on top of. For example, the `node:16-alpine` runtime for a `NodeJS` application.

```Dockerfile
# /Dockerfile

FROM node:16-alpine
# Image node:16-alpine cached for further builds

COPY . .

RUN npm install
RUN npm run build
CMD npm start
```

Taking the above example, the `node:16-alpine` image is cached for further builds. However, Docker will always rebuild the remaining layers, noticing that our `.` directory has changed with any new code changes we add to our application. Directory changes within `.` are captured by the `COPY` command, but if no files are changed within `.`, Docker will be able to reuse the cached layer from the previous build.

We can exploit this functionality for when we are not adding or updating dependencies, which is the case for most commits.

```Dockerfile
# Optimised /Dockerfile

# Define the runtime using a base image
FROM node:16-alpine

# Install project dependencies
COPY package.json package.json
COPY package-lock.json package-lock.json
RUN npm install

# -- CHECKPOINT --

# Build the application
COPY . .
RUN npm run build
CMD npm start
```

Structuring the `Dockerfile` this way, we cache our base image `node:16-alpine` and we cache the image layers up until the `CHECKPOINT` that would contain the installed `node_modules` for the application.

When a change is made to the codebase that does not change the contents of `package.json` or `package-lock.json`, Docker will retrieve the cached layers before the `CHECKPOINT`, and build the remaining layers from there.

This technique can be used in a similar fashion for other runtimes, alongside the caching of static assets that are large in size.

[🔗 Source: Building Efficient Dockerfiles - Node.js](http://bitjudo.com/blog/2014/03/13/building-efficient-dockerfiles-node-dot-js/)