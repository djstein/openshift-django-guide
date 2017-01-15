# RedHat OpenShift Django Deployment Guide

__WARNING: CURRENTLY THIS IS NOT PRODUCTION READY !!!UPDATES COMING!!!__

Using Django 1.10 and Python 2.7

# Contents
1. [Introduction](#Introduction)
2. [Instructions](#Instructions)
3. [Addition of Vue.js Project](#Addition_of_Vue_js_Project)

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
6. [Configure wsgi.py](#Configure_wsgi_py)
7. [Configure Static Files](#Configure_Static_Files)
8. [Allowed Hosts](#Allowed_Hosts)
9. [Pushing Code](#Pushing_Code)

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
  <app>/
  manage.py
  <project_name>/
  requirements.txt
  setup.py
  wsgi.py
```
More files may be required, and for this example the directory 'static' at repo level be used to server static files.

## 4. Database Settings<a name="Database_Settings"></a>
In the\`<project_name>/settings.py change the database connection settings to use OpenShift environment variables.
```python
# <project_name>/settings.py

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

## 5. Configure wsgi.py<a name="Configure_wsgi_py"></a>
Replace the exisiting contents of the wsgi.py with the following:
```python
# wsgi.py (root directory)
#!/usr/bin/python
import os, sys

sys.path.append(os.path.join(os.environ['OPENSHIFT_REPO_DIR']))

os.environ['DJANGO_SETTINGS_MODULE'] = '<project_name>.settings'

virtenv = os.environ['OPENSHIFT_PYTHON_DIR'] + '/virtenv/'

os.environ['PYTHON_EGG_CACHE'] = os.path.join(virtenv, 'lib/python2.7/site-packages')

virtualenv = os.path.join(virtenv, 'bin/activate_this.py')
# This activation of the virtualenv is different for Python 2 and 3, the v2 is shown below
try:
    execfile(virtualenv, dict(__file__=virtualenv))
except IOError:
    pass

#
# IMPORTANT: Put any additional includes below this line.  If placed above this
# line, it's possible required libraries won't be in your searchable path
#
from <project_name>.wsgi import application
```

## 6. Create Action Hook: deploy
Create an OpenShift action hook that will run a set of commands during deployment.
Create this file in the folder, then add permissions to execute.
```bash
touch .openshift/action_hooks/deploy
chmod +x .openshift/action_hooks/deploy 
```
Edit the deploy script to include:
```bash
# .openshift/action_hooks/deploy
cd $OPENSHIFT_REPO_DIR
echo "Executing 'python manage.py migrate'"
python manage.py migrate
echo "Executing 'python manage.py collectstatic --noinput'"
python manage.py collectstatic --noinput
```

## 7. Configure Static Files<a name="Configure_Static_Files"></a>
For this example we will assume the static files are application specific. Thus \<app> has its own static directory.
Ensure that \<app> is included in the <project>/settings.py INSTALLED_APPS. Then add STATIC_ROOT and STATICFILES_DIRS.
```python
# <project_name>/settings.py
INSTALLED_APPS = (
    ...
    'appname',
)

...

STATIC_URL = '/static/'

STATIC_ROOT = os.path.join(BASE_DIR, 'static')

STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'appname/static/'),
]
```

## 8. Allowed Hosts<a name="Allowed_Hosts"></a>
To connect to the application add the allowed hosts for this application, for OpenShift test deployment, add the following
```bash
# <project_name>/settings.py
ALLOWED_HOSTS = [
    ...
    '<app_name>-<domain>.rhcloud.com'
]
```

## 9. Pushing Code<a name="Pushing_Code"></a>
Deploy the codebase to the OpenShift repository by checking the files to be added, ensuring they are correct here we use 'git .', adding a commit message, then pushing.
```bash
git status
git add .
git commit -am "Initial commit"
git push
```

# Addition of Vue.js Project<a name="Addition_of_Vue_js_Project"></a>

This assumes usage of the Vue.js Application template found here:

1. [Django Webpack Loader](#Django_Webpack_Loader)
2. [Webpack Config Changes](#Webpack_Config_Changes)

##1. Django Webpack Loader<a name="Django_Webpack_Loader"></a>
In requirements.txt make the addition of the package to install:
```python
Django
django-webpack-loader
```

Then in \<project_name>/settings.py make additions of loading the module, ensuring the \<vue_app>'s static files are found by collectstatic, and add the settings for Webpack Loader.
```python
# <project_name>/settings.py

INSTALLEED_APPS = [
  'webpack_loader',
]

...

STATICFILES_DIRS = [
    ...
    os.path.join(BASE_DIR, '<vue_app>/static/'),
]

...

WEBPACK_LOADER = {
    'DEFAULT': {
        'BUNDLE_DIR_NAME': 'static/',
        'STATS_FILE': os.path.join(BASE_DIR, '<vue_app>', 'webpack-stats.json')
    }
}
```

## 2. Webpack Config Changes<a name="Webpack_Config_Changes"></a>
```javascript
  entry: [
      'webpack-dev-server/client?http://<app_name>-<domain>.rhcloud.com',
      'webpack/hot/only-dev-server',
      '../src/main'
  ],
  output: {
      path: path.resolve('./static/'),
      filename: "[name]-[hash].js",
      publicPath: 'http://<app_name>-<domain>.rhcloud.com/static/', // Tell django to use this URL to load packages and not use STATIC_URL + bundle_name
  },
```

## 3. Webpack Building<a name="Webpack_Building"></a>
In the \<vue_app> directory, run the script
```bash
npm run build
```
This will automatically bundle the CSS, JavaScript, and other static files of the Vue.js application and place them in appropriate folders that collectstatic will look in.

Unforunately this script cannot be run and bundles on the OpenShift server due to permissions and ability to install other applications not associated with the cartridge.

## 4. Deploy
Once the build has completed, add the new build found in \<vue_app>/static/ to a git commit and push the new changes.
