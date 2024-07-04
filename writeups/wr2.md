---
title: 'Setting Up a Rails Development Environment Using Docker and Docker Compose'
date: 2024-07-04T08:34:37+08:00
draft: false
type: leetcode
tags: ['environment', 'rails 6.1', 'docker', 'ruby 3.1', 'noLocalInstall']
---

Since I've began using Windows OS, I discovered how painful it is to setup development environments at times, even using WSL2. Also at the time I had an internship where I was handed the task to research how to scale a website to make sure that it becomes highly-available. This is when I began to have true first-hand experience with Docker and Docker Compose. From studying about Docker and its capabilities, I began to wonder if it would be possible to set up all my development environments using Docker containers. 

Since the website that I had to scale up at the time was a Ruby on Rails website (namely the Redmine web application), I was doing such research in the context of setting up a Ruby development environment using Docker containers (I found Ruby and Rails even more painful to setup than some other languages and frameworks I've used in the past). This is when I first found the book __Docker for Rails Developers__ by Rob Isenberg. The following post will be about the process that I used to setup my no-local-install Rails development environment using what I learnt from the book, as well as some other processes that I had to follow to get the environment working with databases except for Postgres and the specific versions of Ruby and Rails I was using.

Before starting the post, make sure to have installed Docker and Docker Compose first on your local computer.

## Initializing a Rails Application Without Local Installation of Ruby and Rails

To initialize a new Rails application on my local machine, I didn't have to install Ruby or Rails locally as long as I use Docker, which already has an official image of the Ruby language for version 3.1. The process I followed to initialize a new Rails application without any local installation of dependencies is given below.

1. Enter the directory where you wish to store your project using your terminal.

2. Execute the following command in your terminal to run and enter the container.

    ```bash
    docker run -it --rm -v ${PWD}:/usr/src/app ruby:3.1 bash
    ```

    Once you enter the command, you will see a terminal prompt that looks something like:

    ```bash
    root@[a-bunch-of-letters-and-digits]:/#
    ```

    This would indicate that you have successfully entered the container and will now be executing commands in the container.

3. First, install the latest version of `node` and `yarn` by running the following commands.

    ```bash
    apt-get update -yqq && apt-get install -yqq --no-install-recommends nodejs && \
    curl -fsSL https://raw.githubusercontent.com/tophat/yvm/master/scripts/install.sh | bash && \
    source /root/.yvm/yvm.sh && \
    yvm use 1 
    ```
    This is necessary for setting up Rails v6.1 since there will be major issues later on with webpacker setup if we do not have `yarn` installed.  

4. Now change directories by doing `cd /usr/src/app` (declared above after the `-v` flag) and install the Rails gem. Since I was practicing setting up a Rails development environment for Redmine, which uses specific versions of Rails, here I will be specifying the Rails version while installing the gem.

    ```bash
    gem install rails -v 6.1
    ```

5. Now that Rails is installed, run the following command to generate a new standard full-stack Rails project.
    ```bash
    rails new myapp --skip-test
    ```

    Note that although the container will be removed as soon as we've exited this shell, we do not want to skip the `bundle install` process because we want to generate the webpacker file for our Rails project on the local machine.

6. Once we've generated a new project, type `exit` to exit the container, and we would've returned to our local directory. You can type `ls` to see that the generated directory is actually reflected in our local environment! We have succesfully generated a new Rails project without installing anything except for Docker. 

7. **(Linux Users)** The newly generated project is owned by root on our local machine, since by default containers run as root. Make sure to run the following command to change the files' owner so that the files could actually be modified.

    ```bash
    sudo chown <username>[:<group>] -R myapp
    ```

    **Note:** `[:<group>]` means that this is optional in the above command. 

## Build an Image for the Rails Application

A new image can be built by defining it with a `Dockerfile`. This is done by adding a `Dockerfile` to the root of the project.

1. Make sure that you are in the project root for the following steps. 
2. Create a new file and name it `Dockerfile`.
3. Copy and paste the following contents into the `Dockerfile`. This following piece of code is modified from what was presented in the book since we are using Rails 6.1.
    ```dockerfile
    FROM ruby:3.1

    LABEL maintainer="<your-email>"

    SHELL ["/bin/bash", "-c"]

    RUN apt-get update -yqq && apt-get install -yqq --no-install-recommends nodejs && \
        curl -fsSL https://raw.githubusercontent.com/tophat/yvm/master/scripts/install.sh | bash && \
        source /root/.yvm/yvm.sh && \
        yvm use 1 

    COPY Gemfile* /usr/src/app/

    WORKDIR /usr/src/app

    RUN bundle install

    COPY . .

    CMD ["bin/rails", "server", "-b", "0.0.0.0"]
    ```

4. Build the image using the following command:

    ```bash
    docker build .
    ```

    You can check that a new command has been added using Docker Desktop or the command `docker images`.

5. Now, a Rails server can be ran using our image by running the following command:

    ```bash
    docker run -p 3000:3000 <image-id>
    ```

    You can check if the Rails server has been started successfully by visiting `http://localhost:3000` in your browser. 


## Adding Databases to the Rails Project with Docker Compose

By default, Rails projects sets up a SQLite database for testing and development. However, it is common to change databases to something like MySQL or Postgres. The following section will present different ways to add databases to the Rails project using Docker Compose. 

1. Create a new file called `docker-compose.yml` in the root of your Rails project.

### Postgres

2. Add the Postgres gem to our Rails application by adding `gem 'pg', '~> 1.0'` to the Gemfile. 

3. Add Postgres to the `docker-compose.yml` file. This can be done in two ways:
    1. Hardcode the database environment variables into the `docker-compose.yml` file.

        ```yaml
        version: '3'
        
        services:
            app:
                build: .
                ports:
                    - "3000:3000"
                volumes:
                    - ".:/usr/src/app"
            database:
                image: postgres
                environment: 
                    POSTGRES_USER: <user-name>
                    POSTGRES_PASSWORD: <password>
                    POSTGRES_DB: app_dev
        ```

    2. Setup environment variables in the Rails project.
        1. In the root of the Rails project, create the folders and files `.env/development/app` and `.env/development/postgres` (naming is arbitrarily picked). 
        2. In `.env/development/app`, add the following line:

            ```toml
            DATABASE_HOST=database
            ```

            In `.env/development/postgres`, add the following lines:

            ```toml
            POSTGRES_USER=<user-name>
            POSTGRES_PASSWORD=<password>
            POSTGRES_DB=app_dev
            ```
    
    3. Change the above `docker-compose.yml` file to use the defined environment variables as follows.

        ```yaml
        version: '3'

        services:
            app:
                build: .
                ports:
                    - "3000:3000"
                volumes:
                    - ".:/usr/src/app"
                depends_on:
                    - database
                links:
                    - database
                env_file:
                    - .env/development/app
                    - .env/development/postgres
            database:
                image: postgres
                restart: always
                ports:
                    - "5432:5432"
                env_file:
                    - .env/development/postgres
        ```

4. Modify the `config/database.yml` in the Rails project so that `ActiveRecord` uses Postgres rather than SQLite as its database.

    ```yaml
    default: &default
        adapter: postgresql
        encoding: unicode
        host: <%= ENV.fetch('DATABASE_HOST') %> 
        username: <%= ENV.fetch('POSTGRES_USER') %>
        password: <%= ENV.fetch('POSTGRES_PASSWORD') %>
        database: <%= ENV.fetch('POSTGRES_DB') %>
        pool: 5
        variables:
            statement_timeout: 5000
    
    development:
        <<: *default
    
    test:
        <<: *default
        database: app_test

    production:
        <<: *default
    ```

4. Rebuild the image for the application using the following command:

    ```bash
    docker-compose build
    ```

5. Start Postgres and the Rails server using the following command:

    ```bash
    docker-compose up
    ```

6. You can connect to Postgres from a separate container using the command:

    ```bash
    docker-compose run --rm database psql -U <user-name> -h database
    ```

    Then enter the password `<password>` set in the `docker-compose.yml` file to log in to the database.

### MySQL

2. Add the MySQL gem to our Rails application by adding `gem mysql2` to the Gemfile. 

3. Add MySQL to the `docker-compose.yml` file by creating `.env/development/mysql` where you place the environment variables for the MySQL database and then use those references in the yaml file.

    ```toml
    MYSQL_ROOT_PASSWORD=rootpw
    MYSQL_DATABASE=app_development
    MYSQL_USER=user
    MYSQL_PASSWORD=helloworld
    ```

    ```yaml
    version: '3'

    services:
    app:
        build: .
        ports:
            - "3000:3000"
        volumes:
            - ".:/usr/src/app"
        depends_on:
            - database
        links:
            - database
        env_file:
            - .env/development/app
            - .env/development/mysql
    database:
        image: mysql
        restart: always
        ports:
            - "3306:3306"
        env_file:
            - .env/development/mysql
    ```

    Alternatively, instead of an environment variable file you could also directly hardcode those variables using `environment` in place of `env_file`. 

4. Modify the `config/database.yml` in the Rails project so that `ActiveRecord` uses MySQL rather than SQLite as its database.

    ```yaml
    default: &default
    adapter: mysql2
    encoding: utf8
    host: <%= ENV.fetch('DATABASE_HOST') %> 
    username: <%= ENV.fetch('MYSQL_USER') %>
    password: <%= ENV.fetch('MYSQL_PASSWORD') %>
    database: <%= ENV.fetch('MYSQL_DATABASE') %>
    pool: 5

    development:
        <<: *default

    test:
        <<: *default
        database: app_test

    production:
        <<: *default
    ```

4. Rebuild the image for the application using the following command:

    ```bash
    docker-compose build
    ```

5. Start MySQL and the Rails server using the following command:

    ```bash
    docker-compose up
    ```