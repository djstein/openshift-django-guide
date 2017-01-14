# RedHat OpenShift Django Deployment Guide

Using Django 1.10 and Python 2.7

# Contents
1. [Introduction](#Introduction)
2. [Instructions](#Instructions)

# Introduction<a name="Introduction"></a>

This is a set by step guide for deploying a Django web application to OpenShift in 2017. Unforunately the OpenShift team has stopped major development on version 2 of OpenShift*, and its version 3 which will use Docker containers has not been deployed completely and will not be officially released until the Summer 2017.

*Python 2.7 is used in this guide, as OpenShift only has the avaibale versions of 2.7 or 3.3. Django versions of 1.9 or greater are only availabe on Python 2.7 or 3.5, thus we must continue with 2.7 to make use of the new features. There is a way to great DIY cartridges but they do not play well with the entire OpenShift system / Django deployment process.

# Instructions<a name="Instructions"></a>
Disclaimer this guide assumes you have an OpenShift account and have the OpenShift Client tools known as 'rhc' and the 'git' tools installed and accessabile

## Instruction Contents
1. [Create New OpenShift Application](#Create_New_OpenShift_Application)
2. [Connect PostrgresSQL DB](#Connect_PostrgresSQL_DB)
3. [Add Exisiting Django Project PostrgresSQL DB](#Add_Exisiting_Django_Project)
4. [Connect PostrgresSQL DB](#Connect_PostrgresSQL_DB)
5. [Database Settings](#Database_Settings)

## 1. Create New OpenShift Application<a name="Create_New_OpenShift_Application"></a>
In the terminal:
Create a new OpenShift Application with Python 2.7:
```bash
rhc create-app -a <app-name> -t python-2.7
```
\<app-name> for this tutorial will be 'ostest'
Or on the OpenShift Web Console:
Create the application, enter the app-name, and after the creation, copy the Source Code ssh link and git clone it to your working directory.
```bash
git clone ssh ssh://<hash>@<app-name>-<domain>.rhcloud.com/~/git/<app-name>.git/
```
## 2. Connect PostrgresSQL DB<a name="Connect_PostrgresSQL_DB"></a>
Then connect a PostgreSQL database to the application. Take note of the credentials displayed on output.
```bash
rhc cartridge add -c postgresql-9.2 -a <app-name>
```

## 3. Add Exisiting Django Project<a name="Add_Exisiting_Django_Project"></a>
The current directory structure should include:
```bash
<app-name>/
  .git/
  .openshift/
  requirements.txt
  setup.py
  wsgi.py
```

Now add your Django project to the \<app-name> directory. Assuming standard Django project the directory should look similar:
```bash
<app-name>/
  .git/
  .gitignore
  .openshift/
  LICENSE
  README.md
  appname/
  manage.py
  project/
  requirements.txt
  setup.py
  wsgi.py
```
More files may be required, and for this example the directory 'static' at repo level be used to server static files.

## 4. Database Settings<a name="Database_Settings"></a>
In the project/settings.py change the database connection settings to use OpenShift environment variables.
```python

...
import urlparse

...

db_url = urlparse.urlparse(os.environ.get('OPENSHIFT_POSTGRESQL_DB_URL'))

DATABASES = {'default': {
    'ENGINE': 'django.db.backends.postgresql_psycopg2',
    'NAME': os.environ['OPENSHIFT_APP_NAME'],
    'USER': db_url.username,
    'PASSWORD': db_url.password,
    'HOST': db_url.hostname,
    'PORT': db_url.port,
    }
}
```

