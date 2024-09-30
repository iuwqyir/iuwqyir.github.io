---
title: How to Reduce Docker Image Size
date: 2024-06-03 16:00:00 +0300
categories: [Devops]
tags: [docker, infrastructure, optimization]
robots: index
---

Using Docker images to run your code is a great way to make it portable, scalable and consistent.
You should always to try keep your images as small as possible so that they use less network and disk space and are faster to build, push and pull. Even small improvements in the build process can lead to faster feedback times for developers.

Here are ways how to reduce docker image size:
- **Use the smallest possible base image**
- **Minimize the number of image layers**
- **Implement multi-stage builds**
- **Make sure to exclude unnecessary files**
- **Squash layers**

## Use the Smallest Possible Base Image
It is best practice to start with a minimalist base image, like the `alpine` or `busybox` versions, and only move to a larger image if you need the features it provides.  

```
FROM node:20-alpine
...
```
{: file="Dockerfile" }

If we compare the size of the `node:20` and `node:20-alpine` images, we can see that the `alpine` version is more than 5 times smaller.

![Node Alpine image is 5 times smaller](/assets/img/articles/node-alpine-comparison.png)
_Alpine version of the image is 5 times smaller_

A smaller image is more secure as well because the attack surface is smaller.

## Minimize the Number of Image Layers
Every line in a Dockerfile will create a new layer. Every layer will produce some overhead, so it is best to keep only what you need.  

If you run `docker history $NAME_OF_BUILT_IMAGE` you will see the layers that the image consists of and the size of each layer. This will give you a good idea of where you can cut down on the size.

The bigger impact to build time comes from layer caching. If you make a change to the Dockerfile, docker will recreate all the downstream layers from the layer that was changed. That is why it is better to define dependencies and more consistent actions at the top of the Dockerfile. This is especially important for pipelines with frequent image builds.

## Implement Multi-Stage Builds
Multi-stage builds allow you to have many build steps and then define in the last step what you actually need in the final image.  
For example, if you need a larger base image for building, then you can do that in stage one and copy the resulting build files to the last stage with a smaller base image.

Here is an example using multi-stage builds for a NodeJS application.  

```
############### Stage 1 ###############
FROM node:20-alpine

# Create build directory
WORKDIR /usr/src/build

# Install all dependencies
COPY package*.json ./

RUN npm ci
 
COPY . ./

# Build the application
RUN npm run build

############### Stage 2 ###############

FROM node:20-alpine

# Create app directory
WORKDIR /usr/src/app

# Only install production dependencies and do not execute post and pre install scripts
COPY package*.json ./
RUN npm ci --omit=dev --ignore-scripts

# Copy build files from Stage 1
COPY --from=0 /usr/src/build/dist ./dist

EXPOSE 8000

CMD [ "npm", "run", "start" ]
```
{: file="Dockerfile" }

In Stage 1 all the production and development dependencies are installed and the application is built.  
In Stage 2, only production level dependencies are installed and the built application files are copied from Stage 1 to include only what is necessary.

## Make Sure to Exclude Unnecessary Files
There are many files in a repository that are not really necessary for the production version of the code. These may include development dependencies, CI/CD definitions, scripts, version control and other configuration files.
To make sure these are not included in the image, use a `.dockerignore` file.
```
**/node_modules/
npm-debug.log
.prettierignore
.prettierrc.yml
babel.config.js
commitlint.config.js
jest.config.ts
.eslintrc.yml
.husky
.github
.vscode
.git
coverage
```
{: file=".dockerignore" }

## Squash Layers
Docker has an [experimental `--squash` option](https://docs.docker.com/reference/cli/docker/image/build/#squash){:target="_blank"} when building images. This will squash layers into a new image with a single layer and thus _may produce a smaller image_, especially when multiple layers are modifying the same files.
However, **use this cautiously** as it is an experimental feature and may affect builds negatively. There are also a few downsides to having a single layer, depending on your use-case.
* docker will not be able to take advantage of parallelization when pulling the image
* a lot more space may be used due to storing the build cache version of the image and the squashed version

---
<br>
Overall, docker image size is definitely something you should pay attention to. Even small improvements can have exponential effects.  

Smaller image -> quicker build times -> faster feedback-loop for developers -> less development time.

I have used these techniques to successfully reduce an image's size multiple times. Let me know about your most significant docker image improvement!
