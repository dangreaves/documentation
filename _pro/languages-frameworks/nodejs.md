---
title: Node.js on Docker
weight: 48
tags:
  - node.js
  - languages
  - docker
category: Languages &amp; Frameworks
redirect_from:
  - /docker-integration/nodejs/
---
In this article you will learn about setting up a Node.js-based project on Codeship Pro.

## Services and Steps
Before reading through the documentation please take a look at the [Services]({% link _pro/getting-started/services.md %}) and [Steps]({% link _pro/getting-started/steps.md %}) documentation page so you have a good understanding how services and steps on Codeship work.

## Dockerfile
We will start by creating a `Dockerfile` that lets you run your Node.js based test suite in Codeship.

Please take a look at our [Dockerfile Caching best practices]({{ site.baseurl }}{% link _pro/getting-started/caching.md %}) article first to make sure you build your Dockerfile in a way that only invalidates the Docker cache when necessary.

Following is an example `Dockerfile` with inline comments describing each step in the file.

```Dockerfile
# We're starting from the Node.js 0.12.7 container
FROM node:0.12.7

# INSTALL any further tools you need here so they are cached in the docker build

# Set the WORKDIR to /app so all following commands run in /app
WORKDIR /app

# COPY the package.json and if you use npm shrinkwrap the npm-shrinkwrap.json and
# install npm dependencies before copying the whole code into the container.
COPY package.json ./
COPY npm-shrinkwrap.json ./
RUN npm install

# After installing dependencies copy the whole codebase into the Container to not invalidate the cache before
COPY . ./
```

This Dockerfile will give you a good starting point to install any further packages or tools you might need. Take a look at our [browser testing documentation]({{ site.baseurl }}{% link _pro/continuous-integration/browser-testing.md %}) to find and install any further tools you might need for your build.

## codeship-services.yml

The following example will use the Dockerfile we created to set up a container we call project_name (please change to your specific project name) that will run your build. We're adding a [PostgreSQL container](https://hub.docker.com/_/postgres/) and [Redis container](https://hub.docker.com/_/redis/) so the tests have access to those two services.

When accessing other containers please be aware that those services do not run on localhost, but on a different hostname, e.g. "postgres" or "mysql". If you reference localhost in any of your configuration files you have to change that to point to the hostname of the service you want to access. Setting them through environment variables and using those inside of your configuration files is the cleanest approach to setting up your build environment.

```yaml
project_name:
  build:
    image: organisation_name/project_name
    dockerfile_path: Dockerfile
  # Linking Redis and Postgres into the container
  links:
    - redis
    - postgres
  # Set environment variables to connect to the service you need for your build. Those environment variables can overwrite settings from your configuration files (e.g. database.yml) if configured. Make sure that your environment variables and configuration files work work together as expected.
  environment:
    - DATABASE_URL=postgres://postgres@postgres/YOUR_DATABASE_NAME
    - REDIS_URL=redis://redis
# Service definition that specify a version of the service through container tags
redis:
  image: redis:2.8
postgres:
  image: postgres:9.4
```

For more information about other services you can use with Codeship check out our [services and databases documentation]({{ site.baseurl }}{% link _pro/getting-started/services-and-databases.md %}).

## codeship-steps.yml

Now we're going to set up our steps file that contains a parallelized build configuration.

Every step in a build gets its own clean container and linked services. Any setup commands that are necessary to setup a linked container (e.g. database migrations) need to be run for every step. While this duplicates some of the work, it also makes sure your parallelized test commands run completely independently.

In the following example we're running the acceptance and unit test separately to make the build faster. We're setting that up in our package.json file.

```yaml
- name: ci
  type: parallel
  steps:
  - service: project_name
    command: npm run-script test-acceptance
  - service: project_name
    command: npm run-script test-unit
```

And here is the corresponding script section you can put into your `package.json`. In this case we're running Mocha tests, but you can run any other testing tools as well of course.

```json
"scripts": {
  "test-acceptance": "mocha --require test/support/env --reporter spec --bail --check-leaks test/ test/acceptance/",
  "test-unit": "mocha --require test/support/env --reporter spec --bail --check-leaks test/ test/unit/"
}
```
