
## Deployment instructions

### Basic deployment:

* Press the **Clone & Deploy** button above and follow the instructions.
   * The `hasura quickstart` command clones the project repository to your local system and also creates a **free Hasura cluster** where the project will be hosted for free.
   * A git remote (called hasura) is created and initialized with your project directory.
   * `git push hasura master` builds and deploys the project to the created Hasura cluster.
* The php-laravel app is deployed as a microservice called **app**.
   * Run the below command to open your app:
``` shell
 $ hasura microservice open app
```

### Making changes to your source code and deploying

* To make changes to the app, browse to `/microservices/app/src` and edit the php files according to your requirements.
* For example, make changes to `resources/views/welcome.blade.php` to change the landing page.
* Commit the changes, and run `git push hasura master` to deploy the changes.


## View server logs

If the push fails with an error `Updating deployment failed`, or the URL is showing `502 Bad Gateway`/`504 Gateway Timeout`,
follow the instruction on the page and check the logs to see what is going wrong with the microservice:

```bash
# see status of microservice app
$ hasura microservice list

# get logs for app
$ hasura microservice logs app
```

## Adding dependencies

### Add a php dependency

In order use new php package in your app, you can just add it to `src/composer.json` and the git-push or docker build process will
automatically install the package for you. If the `composer install` steps throw some errors in demand of a system dependency,
you can install those by adding it to the `Dockerfile` at the correct place.

```json
# src/composer.json:

{
    "name": "laravel/laravel",
    "description": "The Laravel Framework.",
    "keywords": ["framework", "laravel"],
    "license": "MIT",
    "type": "project",
    "require": {
        "php": ">=7.0.0",
        "fideloper/proxy": "~3.3",
        "laravel/framework": "5.5.*",
        "laravel/tinker": "~1.0"
    },
    "require-dev": {
        "filp/whoops": "~2.0",
        "fzaninotto/faker": "~1.4",
        "mockery/mockery": "~1.0",
        "phpunit/phpunit": "~6.0",
        "symfony/thanks": "^1.0"
    },
    "autoload": {
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "App\\": "app/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "extra": {
        "laravel": {
            "dont-discover": [
            ]
        }
    },
    "scripts": {
        "post-root-package-install": [
            "@php -r \"file_exists('.env') || copy('.env.example', '.env');\""
        ],
        "post-create-project-cmd": [
            "@php artisan key:generate"
        ],
        "post-autoload-dump": [
            "Illuminate\\Foundation\\ComposerScripts::postAutoloadDump",
            "@php artisan package:discover"
        ]
    },
    "config": {
        "preferred-install": "dist",
        "sort-packages": true,
        "optimize-autoloader": true
    }
}
```

### Add a system dependency

The base image used in this boilerplate is [php:7.1.5-apache](https://hub.docker.com/_/php/) debian. Hence, all debian packages are available for installation.
You can add a package by mentioning it in the `Dockerfile` among the existing `apt-get install` packages.

```dockerfile
# Dockerfile

#start with our base image (the foundation) - version 7.1.5
FROM php:7.1.5-apache

#install all the system dependencies and enable PHP modules
RUN apt-get update && apt-get install -y \
      libicu-dev \
      libpq-dev \
      libmcrypt-dev \
      git \
      zip \
      unzip \
    && rm -r /var/lib/apt/lists/* \
    && docker-php-ext-install \
      intl \
      mbstring \
      mcrypt \
      pcntl \
      pdo_pgsql \
      pgsql \
      zip \
      opcache

#install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin/ --filename=composer

#set our application folder as an environment variable
ENV APP_HOME /var/www/html

#change uid and gid of apache to docker user uid/gid
RUN usermod -u 1000 www-data && groupmod -g 1000 www-data

#change the web_root to laravel /var/www/html/public folder
RUN sed -i -e "s/html/html\/public/g" /etc/apache2/sites-enabled/000-default.conf

# enable apache module rewrite
RUN a2enmod rewrite

#copy the composer dependencies only
COPY src/composer.json $APP_HOME/composer.json

# Install all PHP dependencies without the autoloader
RUN composer install --no-ansi --no-dev --no-interaction --no-autoloader

#set the port to 8080
RUN sed -i -e "s/VirtualHost \*:80/VirtualHost *:8080/g" /etc/apache2/sites-enabled/000-default.conf
RUN sed -i -e "s/Listen 80/Listen 8080/g" /etc/apache2/ports.conf

#Copy the app
COPY src $APP_HOME/

# install all PHP dependencies
RUN composer install --no-ansi --no-dev --no-interaction --optimize-autoloader \
            && rm -rf /root/.composer/cache

#change ownership of our applications
RUN chown -R www-data:www-data $APP_HOME

```

## Deploying your existing Laravel app

Read this section if you already have a Laravel app and want to deploy it on Hasura.

- Replace the contents of `src/` directory with your own app's php files.
- Leave `k8s.yaml`, `Dockerfile` and `conf/` as it is.
- Make sure there is already a `composer.json` file present inside the new `src/` indicating all your php dependencies (see [above](#add-a-php-dependency)).
- If there are any system dependencies, add and configure them in `Dockerfile` (see [above](#add-a-system-dependency)).

## Local development

Running your php-laravel code locally works as it usually would.


### Handling dependencies on other microservices
Your laravel app will at some point depend on other microservices like the database,
or Hasura APIs. In this case, when you're developing locally, you'll have to change your
the endpoints you're using. Ideally, you can use an environment variable to switch between
'DEVELOPMENT' or 'PRODUCTION' mode and use different endpoints.

This is something that you're already probably familiar with if you've worked with databases
before.

#### Laravel app running on the cluster (after deployment)
Example endpoints:
```
if (App::environment('production')) {
  $postgres = 'postgres.hasura' #postgres
  $dataUrl  = 'data.hasura'     #Hasura data APIs
}
```

#### Laravel app running locally (during dev or testing)
Example endpoints:
```
else {
  $postgres = 'localhost:5432'   #postgres
  $dataUrl  = 'localhost:9000'   #Hasura data APIs
}
```

And in the background, you will have to expose your Hasura microservices on these ports locally:

```bash
# Access postgres locally
$ hasura microservice port-forward postgres -n hasura --local-port 5432


# Access Hasura data APIs locally
$ hasura microservice port-forward data -n hasura --local-port 9000
```
