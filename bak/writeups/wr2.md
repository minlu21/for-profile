---
title: 'Setting Up a Rails Web App With Various Databases Using Docker and Docker Compose'
date: 2024-07-04T08:34:37+08:00
draft: false
type: leetcode
tags: ['environment', 'rails 6.1', 'docker', 'mysql', 'ruby 3.1', 'noLocalInstall']
---

Since I have begun using Windows OS, I discovered how painful it is to setup development environments at times, even using WSL2. Also at the time I had an internship where I was handed the task to research how to scale a website to make sure that it becomes highly-available. This is when I began to have true first-hand experience with Docker and Docker Compose. From studying about Docker and its capabilities, I began to wonder if it would be possible to set up all my development environments using Docker containers. 

Since the website that I had to scale up at the time was a Ruby on Rails website (namely the Redmine web application), I was doing such research in the context of setting up a Ruby development environment using Docker containers (I found Ruby and Rails even more painful to setup than some other languages and frameworks I've used in the past). This is when I first found the book **__Docker for Rails Developers__** by Rob Isenberg. The following post will be about the process that I used to setup my no-local-install Rails development environment using what I learnt from the book, as well as some other processes that I had to follow to get the environment working with databases except for Postgres and the specific versions of Ruby and Rails I was using.

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

    This indicates that you have successfully entered the container and will now be executing commands in the container.

3. First, install `nodejs v16` and `yarn v1.22` by running the following commands. Version 16 of `nodejs` is used rather than later versions since there appears to be some bugs in later versions which will affect the installation of `webpacker` later on when setting up our Rails application. `yarn` must be installed as well for setting up Rails v6.1 otherwise there will be major issues with webpacker setup later on. 

    ```bash
    apt-get update -yqq && apt-get install -yqq --no-install-recommends apt-transport-https && \
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && \
    source ~/.bashrc && \
    nvm install 16 && \
    nvm use 16 && \
    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
    apt-get update -qq && apt-get install -y yarn
    ```
    
    **Note:** If you are only creating an **__api-only__** Rails 6.1 application, there is no need to install `nodejs` and `yarn` since `webpacker` will not be automatically set up for you during the project-creation process.

4. Now change directories by doing `cd /usr/src/app` and install the Rails gem. Since I was practicing setting up a Rails development environment for Redmine, which uses specific versions of Rails, here I will be specifying the Rails version while installing the gem.

    ```bash
    gem install rails -v 6.1
    ```

5. Now that Rails is installed, run the following command to generate a new standard full-stack Rails project.

    ```bash
    rails new myapp --skip-test
    ```

    Note that although the container will be removed as soon as we've exited this shell, we do not want to skip the `bundle install` process because we want to generate the webpacker file for our Rails project on the local machine.

6. **(Do not skip this step)** `cd` into your newly created project directory and make sure to check if `public/pack/manifest.json` exists in your project directory. If it does not, make sure to run the following commands to generate the file or there will be major issues when building a Docker image of the project later on: 

    ```bash
    yarn add @babel/plugin-proposal-private-methods @babel/plugin-proposal-private-property-in-object --dev && \
    bin/rails webpacker:install && \
    bin/rails webpacker:compile
    ```

7. Once we've generated a new project, type `exit` to exit the container, and we would've returned to our local directory. You can type `ls` to see that the generated directory is actually reflected in our local environment! We have succesfully generated a new Rails project without installing anything directly on our computer except for Docker.

8. **(Linux Users)** The newly generated project is owned by root on our local machine, since by default containers run as root. Make sure to run the following command to change the files' owner so that the files could actually be modified.

    ```bash
    sudo chown <username>[:<group>] -R myapp
    ```

    **Note:** `[:<group>]` means that this is optional in the above command. 

## Building an Image for the Rails Application

A new image can be built by defining it with a `Dockerfile`. This is done by adding a `Dockerfile` to the root of the project.

1. Make sure that you are in the project root for the following steps. 
2. Create a new file and name it `Dockerfile`.
3. Copy and paste the following contents into the `Dockerfile`. This following piece of code is modified from what was presented in the book since we are using Rails 6.1.
    ```dockerfile
    FROM ruby:3.1

    LABEL maintainer="<your-email>"

    SHELL ["/bin/bash", "-c"]

    RUN apt-get update -yqq && apt-get install -yqq --no-install-recommends apt-transport-https && \
        curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && \
        source ~/.bashrc && \
        nvm install 16 && \
        nvm use 16 && \
        curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
        echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
        apt-get update -qq && apt-get install -y yarn
        

    COPY Gemfile* /usr/src/app/

    WORKDIR /usr/src/app

    ENV BUNDLE_PATH /gems

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

    **Note:** With Docker, you can input only first few characters of the image ID and Docker will be able to identify the correct image.

6. Before continuing on, make sure to stop the container created with the last step and remove it. 

## Adding Database Integration to the Rails Project with Docker Compose

By default, Rails projects set up a SQLite database for testing and development. However, it is common to change databases to something like MySQL or Postgres. The following section will present different ways to add databases to the Rails project using Docker Compose. 

1. Create a new file called `docker-compose.yml` in the root of your Rails project.

### Postgres

2. Add the Postgres gem to our Rails application by adding `gem 'pg', '~> 1.0'` to the Gemfile. 

3. Add Postgres to the `docker-compose.yml` file. This can be done in two ways:
    1. In the root of the Rails project, create the folders and files `.env/development/app` and `.env/development/postgres` in your project root (naming is arbitrarily picked). 
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
    3. Create `docker-compose.yml` in your project to use the defined environment variables as follows.

        ```yaml
        version: '3'

        services:
            app:
                build: .
                ports:
                    - "3000:3000"
                volumes:
                    - .:/usr/src/app
                depends_on:
                    - database
                env_file:
                    - .env/development/app 
                    - .env/development/postgres
            
            database:
                image: postgres
                restart: always
                ports:
                    - "3306:3306"
                env_file:
                    - .env/development/postgres
                volumes:
                    - db_data:/var/lib/postgresql/data

        volumes:
            db_data:
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

5. Rebuild the image for the application using the following command:

    ```bash
    docker-compose build
    ```

    Alternatively, you can also just run the command in step 6.

6. Start Postgres and the Rails server using the following command:

    ```bash
    docker-compose up -d --force-recreate app
    ```

7. To check if your database is up and running, you can connect to Postgres with a separate container using the command:

    ```bash
    docker-compose run --rm database psql -U <user-name> -h database -p
    ```

    Then enter the password `<password>` set in the `docker-compose.yml` file to log in to the database.

### MySQL

2. Add the MySQL gem to our Rails application by adding `gem mysql2` to the Gemfile. 

3. Create a `docker-compose.yml` file in your project root. Then add MySQL to the `docker-compose.yml` file by creating `.env/development/mysql` and `.env/development/app` where you place the environment variables for the MySQL database and web application and then use those references in the yaml file.

    ```toml
    # .env/development/app
    DATABASE_HOST=database
    ```

    ```toml
    # .env/development/mysql
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
                - .:/usr/src/app
            depends_on:
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
            volumes:
                - db_data:/var/lib/mysql

    volumes:
        db_data:
    ```

4. Modify the `config/database.yml` in the Rails project so that `ActiveRecord` uses MySQL rather than SQLite as its database.

    ```yaml
    default: &default
        adapter: mysql2
        encoding: utf8
        host: <%= ENV.fetch('DATABASE_HOST') %> 
        username: root
        password: <%= ENV.fetch('MYSQL_ROOT_PASSWORD') %>
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

5. Rebuild the image for the application using the following command:

    ```bash
    docker-compose build
    ```

6. Start MySQL and the Rails server using the following command:

    ```bash
    docker-compose up -d --force-recreate app
    ```

7. You can connect to MySQL with a separate container using the command:

    ```bash
    docker-compose run --rm database mysql -u root -h database -p
    ```

    Then enter the password `<password>` set in the `docker-compose.yml` file to log in to the database.

### Oracle DB

2. Oracle DB requires installing some more packages to use. Follow the  directions below before proceeding to next steps: 
    1. Go to [this link](https://www.oracle.com/tw/database/technologies/instant-client/linux-x86-64-downloads.html) to download version 12.2.0.1.0 of Instant Client BasicLite, Instant Client SDK, and Instant Client SQLPlus.
    2. Move the zip files from the download into the `vendor/` directory of your Rails project.

3. Add the gems for the Oracle DB adapter into the Gemfile.

    ```bash
    gem 'activerecord-oracle_enhanced-adapter', '~> 6.1', '>= 6.1.6'
    gem 'ruby-oci8'
    ```

4. Edit your `Dockerfile` as follows:

    ```dockerfile
    FROM ruby:3.1

    LABEL maintainer="luming2k10@gmail.com"

    SHELL ["/bin/bash", "-c"]

    RUN apt-get update -yqq && apt-get install -yqq --no-install-recommends build-essential apt-transport-https unzip libaio1 && \
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && \
    source ~/.bashrc && \
    nvm install 16 && \
    nvm use 16 && \
    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
    apt-get update -yqq && apt-get install -yqq yarn

    ENV APP_HOME /usr/src/app
    ENV ORACLE_HOME /opt/oracle
    ENV LD_LIBRARY_PATH /opt/oracle/instantclient_12_2
    ENV BUNDLE_PATH /gems

    COPY vendor/ ${APP_HOME}
    COPY Gemfile* ${APP_HOME}

    RUN mkdir -p ${ORACLE_HOME}

    WORKDIR ${ORACLE_HOME}

    RUN unzip ${APP_HOME}/instantclient-basiclite-linux.x64-12.2.0.1.0.zip && \
        unzip ${APP_HOME}/instantclient-sdk-linux.x64-12.2.0.1.0.zip && \
        unzip ${APP_HOME}/instantclient-sqlplus-linux.x64-12.2.0.1.0.zip

    WORKDIR ${APP_HOME}

    RUN ln -s ${ORACLE_HOME}/instantclient_12_2/libclntsh.so.12.1 ${ORACLE_HOME}/instantclient_12_2/libclntsh.so && \
        ln -s ${ORACLE_HOME}/instantclient_12_2/libocci.so.12.1 ${ORACLE_HOME}/instantclient_12_2/libocci.so

    RUN bundle install

    COPY . .

    CMD ["bin/rails", "server", "-b", "0.0.0.0"]
    ```

5. Create a `docker-compose.yml` file in your project root. Then add Oracle DB to the `docker-compose.yml` file by creating `.env/development/oracledb` and `.env/development/app` where you place the environment variables for the Oracle database and web application and then use those references in the yaml file.

    ```toml
    # .env/development/app
    DATABASE_HOST=database
    ```

    ```toml
    # .env/development/oracledb
    ORACLE_RANDOM_PASSWORD=true
    APP_USER=dbuser
    APP_USER_PASSWORD=helloworld
    ORACLE_DATABASE=APPDB1,APPDB2
    ```

    **Note:** The above configuration will create two databases `APPDB1` and `APPDB2`. You can specify more databases to be created or reduce the number of databases just by adding to or removing from the list of database names.

    ```yaml
    version: '3'

    services:
        app:
            build: .
            ports:
                - "3000:3000"
            volumes:
                - .:/usr/src/app
            depends_on:
                - database
            env_file:
                - .env/development/app 
                - .env/development/oracledb
        
        database:
            image: gvenzl/oracle-free
            ports:
                - "1521:1521"
            env_file:
                - .env/development/oracledb
            volumes:
                - db_data:/opt/oracle/oradata

    volumes:
        db_data:
    ```


## Setting Up the Database

Create the development and test databases using the standard Rails command `bin/rails db:create`. Since we are using Docker, run the following command:

```bash
docker-compose run --rm app bin/rails db:create
```

**Note (Oracle DB):** It will be necessary to use the password for the `SYS` user (i.e., the root user) when we create the database. You can either set the `ORACLE_PASSWORD` to your own password or obtain the random password (assuming `ORACLE_RANDOM_PASSWORD` is set to true) from the container logs.

## Afterwards

### Creating a New Model and Migrating Data

Suppose we want to create a new model for `Todos` with the fields of todo item and completion status and automatically generate its new controllers using `rails generate scaffold`. Then we can run the following command:

```bash
docker-compose exec app bin/rails generate scaffold Todo item completion:boolean
```

Then we can migrate the data to the database using the following command:

```bash
docker-compose exec app bin/rails db:migrate
```

Finally, check that the web application and database integration works by visiting `http://localhost:3000/todos`. The final result is shown below.

![todo_web_app](/images/Screenshot%202024-07-08%20092936.png)