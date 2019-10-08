How to Deploy Django Applications on Heroku
===========================================
- This a detailed documentation on deploying `Django+Postgres` project on heroku(free tier).
- You can skip sections that you dont need, use the table of contents below.
- Contributions are welcomed, from spelling mistakes to new/better ways/features, see [contributing](#contributing) section.
## Table Of Contents
- [Assumptions](#assumptions)
- [Setup](#setup)
    - [Installing Heroku CLI](#install-heroku-cli)
    - [Virtual environment](#virtual-environment)
    - [Procfile for heroku](#create-a-procfile)
    - [Using enviromennt variables and why we use them](#environment-variables)
- [Database settings](#database-settings)
- [Static and media files](#static-and-media-files)
- [Specifying a python runtime](#specifying-your-python-runtime)
- [Creating Heroku App and Deploying Django Project](#creating-heroku-app-and-deploying-django-project)
    - [Adding Configurations to your Heroku app](#adding-configurations-to-your-heroku-app)
    - [Pushing Project To Heroku](#pushing-project-to-heroku)
    - [Migrating The Database]("database-migrations)
    - [Deploying Local Postgres To Heroku](#pushing-local-postgres-database-to-heroku)
    - [Running tests on heroku](#confirming-that-tests-run-on-heroku)
- [Debbing Deployment Errors and Common issues](#why-am-i-getting-errors)
- [Contributing](#contributing)
- [Resources](#resources)


## Assumptions
* You have a django project that you want to deploy.
* If using a database, it's Postgres (At the time of writing Heroku only supported Postgres).
* You are working on a virtual env. See this section to see how to set up one.

### Tested django versions
* django 1.11
* django 2.2

## Checklist
- [ ] Procfile.
- [ ] Add settings to .env file.
- [ ] Set up database url.
- [ ] Configure Whitenoise for static files
- [ ] Create a requirements.txt file.
- [ ] Create a runtime.txt file to tell heroku which python runtime to use.
- [ ] Create heroku app and a postgres instance.
- [ ] Deploy.

## Setup
### Install heroku CLI
Create An account on heroku if you dont have one, [Sign up](https://signup.heroku.com/).

Then install the [Heroku Toolbelt](https://toolbelt.heroku.com/). It is a command line tool to manage your Heroku apps

After installing the Heroku Toolbelt, open a terminal and login to your account:
```bash
$ heroku login
```

### Virtual environment
- You'll need to be working on a virtual environment
```bash
cd projectdir
python3.6 -m venv virtual
source virtual/bin/activate
```
- You can then install your required libraries, for example to install django
```bash
pip install django
```

### Create a Procfile
Heroku apps include a `Procfile` that specifies the commands that are executed by the app’s dynos. 

For more information read on the [heroku documentation](https://devcenter.heroku.com/articles/procfile).

You'll also need to install gunicorn, a Python WSGI HTTP Server as our production server. 

```bash
pip install  gunicorn 
```

Create a file named `Procfile` in the project root with the following content:
```
web: gunicorn your_project_name.wsgi --log-file -
```

### Settings
- First thing we want to do is to move our settings to a `.env` 


### Environment Variables
- If you dont get why we use environment variables may be this can explain.
#### Why environment variables
- The primary use case for environment variables is to limit the need to modify and re-release an application due to changes in configuration data.
- Modifying and releasing application code is relatively complicated and increases the risk of introducing undesirable side effects into production.

#### using .env
- Create a `.env` file at the root of your project.
- Next lets install libary that will enable us to easily read environment variables
```bash
pip install python-decouple
```
- We will move the following settings in django to our env
```
SECRET_KEY
DEBUG
ALLOWED_HOSTS
```
- Below is an example of how you would use decouple to read an env variable
```python
from decouple import config
SECRET_KEY = config('SECRET_KEY')
print(SECRET_KEY)
```

- To use decouple we need to import it our `settings.py` file in django, and replace the generated settings with: 
```python 
# settings.py

# first import decouple
from decouple import config, Cast
...
#secret_key
SECRET_KEY = config('SECRET_KEY')

#debug
DEBUG=  config('DEBUG', cast=bool)

#allowed_hosts
ALLOWED_HOSTS = config('ALLOWED_HOSTS',cast=Csv())
```

- Then add the following to your `.env` file.
```bash
SECRET_KEY='YOUR SUPER SECRET KEY'
ALLOWED_HOSTS='127.0.0.1,localhost,.herokuapp.com'
DEBUG='True'
```

### Database settings
- If using a database on heroku, you'll need to use postgres(at the time of writing this it was the officially supported relational database).
- We will use a library `dj-database-url` to generate the database settings from a database url, meaning we dont have to specify the engine and database address, username, password etc in the settings.py file.
- We also need to install `psycopg2` our postgres engine.
```bash
pip install dj-database-url psycopg2-binary
```
- Then add the database url to your `.env` file in the following format `postgres://<username>:<password>@<address>:<port>/<database_name>`, see example below
```bash
DATABASE_URL='postgres://jak:1234@localhost:5432/students'
```
- Then replace your database settings in `settings.py` with
```python
# first remember to import dj-database url at the top of settings.py file
import dj_database_url
...
DATABASES = {
    'default': dj_database_url.config(
        default=config('DATABASE_URL')
    )
}
...
```
- You'll need to migrate to create the tables on your local database if you were not using postgres before

### Static and media files
>> Django does not support serving static files in production. However, the fantastic WhiteNoise project can integrate into your Django application, and was designed with exactly this purpose in mind.
- Lets install whitenoise
```bash
pip install whitenoise
```
- Remember to update requirements.txt `pip freeze > requirements.txt`.
- Add whitenoise to your django app by adding it as a middleware in `settings.py`
```python
#settings.py
MIDDLEWARE_CLASSES = (
    'whitenoise.middleware.WhiteNoiseMiddleware',
    ...
```
- Then add this settings in for django static files and media files(optional), in `settings.py`. Most likely at the bottom of your `settings.py`, `STATIC_URL` will already be there.
```python
#settings.py

# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/1.9/howto/static-files/
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
# Extra places for collectstatic to find static files.
STATICFILES_DIRS = (
    os.path.join(BASE_DIR, 'static'),
)

#media
MEDIA_URL = '/media'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```
- **optional** If you like to enable file compressions you call also add this setting, to enable whitenoise to use gzip.
```python
#settings.py 
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'
```

## Specifying your python runtime

- This file contains the python version you are using for heroku to use.
- create `runtime.txt` in your project root and add your python version in the following format
```
python-3.6.8
```
- You can find a list of supported python runtimes on heroku here, [Heroku Python Runtimes](https://devcenter.heroku.com/articles/python-runtimes).


## Creating Heroku App and Deploying Django Project.

- At the root directory of your project(where there's `manage.py`)

- Next create the heroku app, make sure you are already logged in to heroku and have the heroku cli installed if not see step 1 of this doc.
```bash
heroku create your-app-name
```
- Create a postgres addon to your heroku app.(optional for those intending to use another database provider)
```bash
heroku addons:create heroku-postgresql:hobby-dev
```
### Adding Configurations to your Heroku app
- We need to set a way to set our environment variables on heroku, below is a list of configurations we will set, `DATABASE_URL` is already set when we added the heroku postres addon.
```bash
SECRET_KEY='YOUR SUPER SECRET KEY'
ALLOWED_HOSTS='127.0.0.1,localhost,.herokuapp.com'
DEBUG='True'
# to disbale heroku from running collect static when push our project
DISABLE_COLLECTSTATIC=1
# dont add this, unless you plan on using an external database its already generate
DATABASE_URL='generated database url, dont add this'
```
- Theres two ways(that I know) you can use to add configurations/settings to your heroku app

#### Via the Dashboard
- Log in to your [heroku dashboard](https://dashboard.heroku.com/apps), select your app and  go to the settings tab. Click on the Settings menu and then on the button Reveal Config Vars:
- Next add all the environment vaiables, by default you should have `DATABASE_URI` configuration created after installing postgres to heroku.
- The environment variables to add are the same as the ones in your `.env` file, picture below shows an example:

<img src="https://i.imgur.com/2Wi41Vq.png" alt="Heroku dashboard" width="400" height="300">

#### Via Heroku CLI
- Alternatively you can use the heroku cli to set environment variabes, example to set the `SECRET_KEY`
```bash
heroku config:set SECRET_KEY=supersecretkey
```
- You can use the command below to set ones in your `.env` but for some reason on my machine it skips the `SECRET_KEY` so i have to set it manually using the command above.
```bash
heroku config:set $(cat .env | sed '/^$/d; /#[[:print:]]*$/d')
```

- Remember to confirm that you have set all your environment variables before you continue.

### Pushing Project to heroku
- First confirm that your project works as expected on your local before pushing. If it does not work on your local environment it will most likely not work on heroku too.
- Next update your `requirements.txt` if you haven't, `pip freeze > requirements.txt`, check if the file has a package called `pkg-resources` and remove it.
- Commit all your changes with git.
```bash
git push heroku master
```
- If you did everything correctly then the deployment should be done.

### Database Migrations 
- If you plan on pushing your local database to heroku, skip this step
```bash
heroku run python manage.py migrate
```
### Pushing local postgres database to heroku
- If you instead wish to push your postgres database data to heroku then run
```bash
heroku pg:push mylocaldb DATABASE_URI --app yourappname
```
- Where `mylocaldb` is the database url to your local database, same as the one in your local `.env` or your database name if you are running postgres on your machine bare metal.

### Confirming that tests run on heroku
```bash
heroku run python manage.py test
```

- You can now open your app on your browser by visiting `https://yourappname.herokuapp.com`.
- Once your done debbuging remember to change `DEBUG` to `false`.
- If for some reason you get an error, see the [why am i getting errors section](#why-am-i-getting-errors).

## Why Am I getting errors
- To debug application errors on heroku first make sure `DEBUG` is enabled or that you have enabled `logging` in django. Then go through the logs via the admin dashboard or running `heroko logs` in you cli.
- Below are examples of common issues.

### Turning Debug to False gives me a error 500.
- This can be a result of a few things, the most important thing will be for you to pull the logs and debug.
- To enable logging in your app see the offical django docs on [logging](http://docs.djangoproject.com/en/2.2/topics/logging/).
- You can also see this answer on [stack overflow](https://stackoverflow.com/a/52314952/9649617)

### Runtime issues.
- Sometime your app may fail because of different python runtimes, reason is some packages versions are only available on specific python runtimes.
- Make sure that the runtime you use in development is the python runtime you use on heroku. For example I've seen this issue with people who use `python-3.6.8` in development and use `python-3.7.2` on heroku
- See this [section](#specifying-your-python-runtime)

### My images wont show after sometime
-  Heroku doesn't support long term image storage on dynos as per their [documentation](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem). You'll have to host your uploaded files somewhere else (or from the database, which is a bad option or git another bad option).
- The following are options that I've tried and even have python sdk's
    - [Amazon S3](https://aws.amazon.com/s3/)
    - [Cloudinary](https://cloudinary.com/) 
    - [Uploadcare](https://uploadcare.com/)

###

**To add more situations**

## Contributing
- Contributions are welcomed especially if you encounter a common issue and solved it you can add it to the `Why Am I getting errors` section.
- To make a contribution either fork this repo, make your changes and make a pull request or just contact me.

## Resources
### heroku Docs
* [Heroku Postgres](https://devcenter.heroku.com/articles/heroku-postgresql)
* [Static Files On Heroku and Whitenoise](https://devcenter.heroku.com/articles/django-assets)
* [Heroku python Runtimes](https://devcenter.heroku.com/articles/python-runtimes)
* [Heroku Procfile](https://devcenter.heroku.com/articles/procfile)
* [Heroku getting started with python](https://devcenter.heroku.com/articles/getting-started-with-python#introduction)
* [Heroku Deploying python](https://devcenter.heroku.com/articles/deploying-python)
* [Heroku Django App configuration](https://devcenter.heroku.com/articles/django-app-configuration)
* [Heroku Gunicorn](https://devcenter.heroku.com/articles/python-gunicorn)

### Other helpfull articles
* https://simpleisbetterthancomplex.com/tutorial/2016/08/09/how-to-deploy-django-applications-on-heroku.html
* https://simpleisbetterthancomplex.com/2015/11/26/package-of-the-week-python-decouple.html


