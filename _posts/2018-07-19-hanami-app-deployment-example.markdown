---
layout: post
title:  "Deploying Hanami web application with Puma, Nginx and PostgreSQL using Docker"
date:   2018-07-19 21:45:40 +0200
tags:   [hanami, deployment, puma, docker]
---

## Deploying Hanami web application with Puma, Nginx and PostgreSQL using Docker

The steps below describe a deployment process of a [Hanami](http://hanamirb.org) web application. The application was deployed to a VM with Ubuntu 16.04 server.


### 1. Prepare the production environment variables

Prepare the environment variables in the `.env.production` file as shown in the example below.Make sure the file is not under source control as it might end up in the repository. To avoid that scenario add the file to `.gitignore`.

Set the user and password for the postgres role. The `localhost` in the database url can be replaced with a reference to the postgres Docker container.

The web sessions secret can be generated with `hanami secret` since version 0.8. For previous versions we can use the command line and execute `ruby -rsecurerandom -e "puts SecureRandom.hex(64)"` which will generate and print the secret.

Example of `.env.production`:
```ruby
DATABASE_URL="postgres://user:pass@localhost:5432/db_name"
WEB_SESSIONS_SECRET="some_secret"
SMTP_USERNAME="tbd"
SMTP_PASSWORD="tbd"
SERVE_STATIC_ASSETS="true"
```


### 2. Setup Puma and PG
In order to use Puma and PostgreSQL in production add the respective gems to the gemfile in the production group. They will be installed during the Docker build process along everything else in the gemfile.

It is possible to have a separate configuration for Puma. To do that see the guides on the [github page](https://github.com/puma/puma).


### 3. Install Docker
In order to use the Docker for this deployment procedure first install the [Docker Engine](https://docker.github.io/engine/installation/) and the [Docker Compose](https://docs.docker.com/compose/install/).


### 4. Prepare the Dockerfile and Compose files
Here are the examples of the files used for deployment.

**Dockerfile:**

```ruby
FROM ruby:2.3.0

# install cron
RUN apt-get update && apt-get install cron -y

# throw errors if Gemfile has been modified since Gemfile.lock
RUN bundle config --global frozen 1

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install program to configure locales
RUN apt-get install -y locales
RUN dpkg-reconfigure locales && \
  locale-gen C.UTF-8 && \
  /usr/sbin/update-locale LANG=C.UTF-8

# Install needed default locale for Makefly
RUN echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen && \
  locale-gen

COPY Gemfile /usr/src/app/
COPY Gemfile.lock /usr/src/app/
COPY .ruby-version /usr/src/app/
RUN bundle install

COPY . /usr/src/app

ENV LANG=en_US.UTF-8

ENV HANAMI_HOST=0.0.0.0
ENV HANAMI_ENV=production

EXPOSE 2300
CMD ["hanami server"]
```

**Compose file (`docker-compose.yml`):**

```ruby
version: '2'
services:
  postgres:
    image: postgres
  web:
    build: .
    command: hanami server
    ports:
      - 2300:2300
    depends_on:
      - postgres
    links:
      - postgres
```

The Dockerfile specifies the image to be build by starting with the specification of the image to be build on top of. In this case the base image is `ruby:2.3.0` which is the official image on the Docker Hub. The file also contains commands for:

- creating an app folder,
- installing and setting the locales,
- gemfile preparation and bundling,
- copying the source code to the container folder and finally
- running the app with `hanami server` command.


The Compose file defines the required services: one for the web application and another for the database. The web service builds the image from the current application directory based on the Dockerfile, forwards the port 2300 on the container to the port 2300 on the host machine and defines the link to the postgres container.

The postgres service uses the latest public image on the Docker Hub.


### 5. Pull the repository
After all the Docker related files are prepared it's time to push the final commits to the repo and then pull the source code from the repository to the server VM.

**After a successful pull any untracked files needed for the app must be copied manually to the app folder on the server.**

In my case, these were some log files, the `.env.production` file and some assets.


### 6. Run Docker commands
To build the images and run containers we use the following docker commands in the application root folder:

**In case of an error when running any of the docker commands, run them as sudo.**

```ruby
docker-compose build
docker-compose up -d
```

The second command will start both containers as background services (hence the `-d` flag). Check the built images and containers with following commands:

* Display all images: `docker images`
* Display running containers: `docker ps`

Use the `-a` flag to show all (not only active) images and containers.


### 7. Create the database in the DB container
After the successful built process and start up of the two containers the next step is to create the database for the application.

This can be done in the interactive shell in the database container. To access the container execute the following command: `docker exec -it name_of_the_container bash`.

Then change the user to postgres user and create the user for the database and the database itself.

```ruby
sudo su postgres
psql
CREATE ROLE some_user SUPERUSER LOGIN;
ALTER USER some_user WITH PASSWORD 'some_password';
CREATE DATABASE db_name;
```

Both `some_user` and `some_password` must match with those in the `.env.production`.

**It is also possible to use the default postgres superuser and specify that in the `.env.production` file.**


### 8. Rake and Assets
If there are any prerequisites related to the database in order to run the application they need to be executed inside the application container via the shell same as above.

In my case I ran the database migration with `hanami db migrate` and some rake tasks to create an admin and some application settings.

Specifically I had to copy the fonts: `cp -r /usr/src/app/apps/web/assets/fonts /usr/src/app/public/`

Finally, I've also precompiled the assets with `hanami assets precompile`.

**Assets need to be precompiled every time the application image is rebuilt.**


### 9. Install and setup Nginx

#### Installation
In my particular case for Ubuntu 16.04 (Xenial) I've executed the commands below to install Nginx.

```ruby
sudo apt-get update
sudo apt-get install nginx
```

For details see [these installation guides](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/).


#### Configuration
As suggested [here](https://www.ghostforbeginners.com/how-to-proxy-port-80-to-2368-for-ghost-with-nginx/) I've removed the default configuration files below...

```ruby
/etc/nginx/sites-available/default
/etc/nginx/sites-enabled/default
/etc/nginx/conf.d/default
```

...  and added my specific configuration file(`my_app.config`) in `/etc/nginx/sites-enabled`:


```ruby
server {
  listen 80;
  server_name domain_name.com;
  access_log /var/log/nginx/my_app.access.log;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;

    proxy_pass http://127.0.0.1:2300/;
    proxy_redirect off;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

}
```

To use this example replace the server_name and access_log names accordingly.

Then reload the configuration with `sudo nginx -s reload`.


### 10. Credits
Special thanks to [vasspilka](https://github.com/vasspilka) for extensive help during the deployment process and especially for preparing the Docker and Nginx configuration.

{% include comments.html %}
