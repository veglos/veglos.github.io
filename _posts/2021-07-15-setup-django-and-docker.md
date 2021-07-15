---
layout: post
title:  "Setup Django and Docker"
categories: howto 
tags: django docker python
---
## Intro

It is possible to develop only with Docker (rather than a local system). Therefore a virtual environment is not necessary since dependencies are always installed in the container. Also, environment variables are fetched with docker-compose.yml, and then again django-environ is not really necessary. However I want to be able to keep both options available, since Docker might not always be available.

Full project available on GitHub: [https://github.com/veglos/django-docker](https://github.com/veglos/django-docker)

## Setup

1. Create the project directory and it's virtual environment.

```shell
mkdir django-celery
cd django-celery

python -m venv venv
venv\Scripts\activate.bat
```

2. Initiate the git repository

```shell
git init
```

3. Create the **.local.env** file for environment variables with the following content:

```shell
DEBUG=1
SECRET_KEY=django-insecure-se@p1rdv@wl1(8jb1f^#(m&)dzm=l+hdot7w!h*$bac19t&ro^
POSTGRES_DB=postgres_db
POSTGRES_USER=postgres_user
POSTGRES_PASSWORD=postgres_pwd
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
```

3. Create the **.docker.env** file for environment variables with the following content:

```shell
DEBUG=1
SECRET_KEY=django-insecure-se@p1rdv@wl1(8jb1f^#(m&)dzm=l+hdot7w!h*$bac19t&ro^
POSTGRES_DB=postgres_db
POSTGRES_USER=postgres_user
POSTGRES_PASSWORD=postgres_pwd
POSTGRES_HOST=db
POSTGRES_PORT=5432
```

>Notice the difference with **.local.env** on POSTGRES_HOST=db. This is because docker will reference the db service in the **docker-compose.yml** file.

4. Create the **.gitignore** file with the following content:

```txt
/venv
.env
```

5. Create the **requirements.txt** file with the following content:

```txt
Django==3.2.4
django-environ==0.4.5
psycopg2==2.9.1
```

6. Install dependencies

```shell
pip install -r requirements.txt
```

7. Create django project 'myproject' and the app 'myapp'

```shell
django-admin startproject myproject .
python manage.py startapp myapp
```

8. Modify myproject/settings.py with:

``` yml
import os
import environ
```

``` yml
# Load and read .local.env file
# OS environment variables take precedence over variables from .local.env
env = environ.Env()
env.read_env(os.path.join(BASE_DIR, '.local.env'))
```

``` yml
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env.bool('DEBUG', False)
```

``` javascript
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env("POSTGRES_DB"),
        'USER': env("POSTGRES_USER"),
        'PASSWORD': env("POSTGRES_PASSWORD"),
        'HOST': env("POSTGRES_HOST"),
        'PORT': env("POSTGRES_PORT"),
    }
}
```

9. Create **Docker** file with the following content:

``` yml
FROM python:3

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED=1

WORKDIR /usr/src/app

COPY requirements.txt /usr/src/app

RUN pip install -r requirements.txt

COPY . /usr/src/app
```

10. Create **docker-compose.yml** with the following content:

``` yml
version: "3.9"
   
services:

  db:
    image: postgres    
    env_file:
      - .docker.env
    ports:
      - "5432:5432"
    volumes:
      - type: volume
        source: app-database
        target: /var/lib/postgresql/data

  app:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/usr/src/app
    env_file:
      - .docker.env
    ports:
      - "8000:8000"
    depends_on:
      - db
volumes:
  app-database:
    name: app-database
```

11. Lauch with docker

``` shell
docker-compose up
``` 

Browse [localhost:8000](http://localhost:8000) to check if the project is up.

Close containers with

``` shell
docker-compose down
``` 

12. Launch from local with only the database in a container (**rememeber to activate the virtual environment if you are using another terminal**)

``` shell
docker-compose up db
python manage.py runserver
```

Browse [localhost:8000](http://localhost:8000) to check if the project is up.