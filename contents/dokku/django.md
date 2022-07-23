# Deploy django to production with dokku

In this document I will report on some important steps to get and maintain your Django project in production using Dokku.

If you want to jump to the finished project, [click here.](https://github.com/Mateus-Brito/smallurl)

## Create a new user

let's start by creating a new user and adding it to a sudo and [dokku](https://dokku.com/docs/deployment/user-management/#granting-other-unix-user-accounts-dokku-access) group so we don't have to call sudo on every command.
```bash
adduser dokku
sudo usermod -a -G sudo dokku
echo "%dokku ALL=(ALL:ALL) NOPASSWD:SETENV: /usr/bin/dokku" > /etc/sudoers.d/dokku-users
sudo usermod -a -G dokku dokku
su dokku
```

## Install and configure the Dokku

Let's install dokku, you can look for a newer version on the [official website.](https://dokku.com/)
```
wget -qO- https://raw.githubusercontent.com/dokku/dokku/v0.27.8/bootstrap.sh | sudo dokku_TAG=v0.27.8 bash
```

Now create an application with any name, I'll call mine `django-dokku`.
I will also add the envs: SECRET_KEY, ALLOWED_HOSTS and DOMAIN.
```bash
sudo dokku apps:create django-dokku
dokku config:set django-dokku PYTHON_ENV=production SECRET_KEY=12345 ALLOWED_HOSTS=your_server,your_domain DOMAIN=https://your_domain
```

If you are installing Nginx for the first time you may still have the nginx welcome page visible, I will remove it with the command below.
```bash
sudo rm /etc/nginx/sites-enabled/default
```

## Postgres (with PostGIS) 

The use of PostGis is indicated for databases that work with geometric and spatial data storage. In this project I'm just using as an example to use a non-default postgres image.

Install the postgres plugin and create a service and link with the app created earlier.
```bash
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git postgres
sudo dokku postgres:create dokku-postgis --image "mdillon/postgis" --image-version "latest"
dokku postgres:link dokku-postgis django-dokku
```

## Generate your SSH key locally

If you already have an SSH key, skip the command below.
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

If everything goes correctly you will have two new files:
```bash
id_rsa
id_rsa.pub
```

Now send your local public key to the server to login to the remote host using public key authentication.
```bash
cat ~/.ssh/id_rsa.pub | ssh dokku@server_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## Configure your django project

create a file called ```runtime.txt``` file on root of project with number of python version;
```bash
python-3.10.5
```

Create a file called ```Procfile```  that specifies the commands that are executed by the application at startup.
It is common to find the migrate command in each release of the application, to show how to run the separate command I have not added it here.

Note: remember to change `django_project_name` to your project name.
```bash
web: gunicorn django_project_name.wsgi --log-file -
```

## Deploying with git

Create a new connection to a remote repository to local git repo and push your django project. 
```bash
git remote add dokku dokku@dokku_server:django-dokku
git push dokku main
```

On the server side, migrate your database and restart your application.
```bash
sudo dokku run django-dokku python manage.py migrate
```

Now you can access your django app via server ip :)

## Configure your domain ( or subdomain )

It's pretty simple, just replace `your_domain` with your domain name. (Just remember to point your domain to your server's ip)
```bash
dokku domains:set django-dokku your_domain
```

SSL certificate makes your website more secure for your customers through encryption. Let's install a plugin to generate and manage self-signed certificates.
Replace `your@email.tld` with your email before running the command below.
```bash
sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
dokku config:set --no-restart django-dokku DOKKU_LETSENCRYPT_EMAIL=your@email.tld
sudo dokku letsencrypt:enable django-dokku
```

Turn on auto-renewal when needed.
```bash
dokku letsencrypt:auto-renew django-dokku
```

## Zero deploying time

By default, Dokku will wait 10 seconds after starting each container before assuming it is up and proceeding with the deploy. [See more here.](https://dokku.com/docs/deployment/zero-downtime-deploys/)

Create a file named ```CHECKS``` to more accurately check whether an application can serve traffic or not.
```bash
:5000/admin My Django project
```

## Continuous Integration and Deploying with Github Actions

Create a file called `deploy.yml` in the `.github/workflows` folder with the contents below. Lembre de substituir `appname` pelo nome do seu aplicativo que chamamos aqui de django-dokku.

Create another key to use during deployment, add the public key as authorized on the server and the private key as a variable called `SSH_PRIVATE_KEY` in the github repository.
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
cat id_ed25519.pub >> ~/.ssh/authorized_keys
```

Make sure you have all the variables listed below:
- DJANGO_SECRET_KEY - Django secret key.
- ALLOWED_HOSTS - A list of strings representing the host/domain names that this Django site can serve.
- DOKKU_HOST - You server ip.
- SSH_PORT - SSH connection port.
- SSH_PRIVATE_KEY - The private key created earlier.

Note: In the 'Deploy to main' task, two conditions are used, the first one if a direct push is made to the main branch and the second one if someone starts the workflow manually through a click on the GUI.
```
name: Deploy to Dokku

on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main
  
  workflow_dispatch:
    inputs:
      branch:
        description: 'Branch'
        required: true
        default: 'main'

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.10.5]
    steps:
    - uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run Tests
      run: |
        python manage.py test
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Cancel Previous Runs
        uses: styfle/cancel-workflow-action@0.9.1
        with:
          access_token: ${{ github.token }}

      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Deploy to main
        if: github.ref == 'refs/heads/main' || (github.event_name == 'workflow_dispatch' && github.event.inputs.branch == 'main')
        uses: dokku/github-action@master
        with:
          git_remote_url: 'ssh://dokku@${{ secrets.DOKKU_HOST }}:${{ secrets.SSH_PORT }}/~/appname'
          ssh_private_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

## Storage on Amazon S3

Create a user using the path below.
```
Services > Security, identity, & Compilance > IAM -> USERS -> Add user
```

Give any name and check `Access key - Programmatic access`. Click em Next to go to Set permissions.
Click in `create group` and select `AmazonS3FullAccess` option to Provides full access to all buckets via the AWS Management Console.

Just click em next (Check info in review page) and save `user`, `Access key ID` and `Secret access key` info.

Now create a bucket using the path below.

Note: Turn off "Block all public access".
```
Services -> Storage -> S3 -> Create a bucket
```

Set your new environment variables:
- AWS_ACCESS_KEY_ID - Your user Access key ID
- AWS_SECRET_ACCESS_KEY - Your user Secret access key
- AWS_STORAGE_BUCKET_NAME - Your storage bucket name

```bash
dokku config:set django-dokku AWS_ACCESS_KEY_ID=123 AWS_SECRET_ACCESS_KEY=123 AWS_STORAGE_BUCKET_NAME=appname-static
```

In the settings.py file I chose to configure `whitenoise` for the non-production environment, and for the production environment the boto3 lib to store on Amazon s3.
```bash
if PYTHON_ENV == "production":
    AWS_ACCESS_KEY_ID = config("AWS_ACCESS_KEY_ID")
    AWS_SECRET_ACCESS_KEY = config("AWS_SECRET_ACCESS_KEY")
    AWS_STORAGE_BUCKET_NAME = config("AWS_STORAGE_BUCKET_NAME")
    AWS_S3_CUSTOM_DOMAIN = f"{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com"
    AWS_S3_OBJECT_PARAMETERS = {
        "CacheControl": "max-age=86400",
    }
    AWS_DEFAULT_ACL = None
    # s3 static settings
    STATIC_LOCATION = "static"
    STATIC_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/{STATIC_LOCATION}/"
    STATICFILES_STORAGE = 'appname.core.storages.StaticStorage'
    STATICFILES_DIRS = (BASE_DIR / "static",)
    # s3 public media settings
    PUBLIC_MEDIA_LOCATION = "media"
    MEDIA_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/{PUBLIC_MEDIA_LOCATION}/"
    DEFAULT_FILE_STORAGE = "appname.core.storages.PublicMediaStorage"
else:
    STATIC_URL = "/static/"
    STATIC_ROOT = BASE_DIR / "staticfiles"
    STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"

    MEDIA_URL = "/media/"
    MEDIA_ROOT = "/storage/media/"
```

Create the file `appname.core.storages` with the content below.

Note: The `file_overwrite` flag when disabled generate a random name to our files.
```python
from django.conf import settings
from storages.backends.s3boto3 import S3Boto3Storage


class StaticStorage(S3Boto3Storage):
    location = settings.STATIC_LOCATION


class PublicMediaStorage(S3Boto3Storage):
    location = settings.PUBLIC_MEDIA_LOCATION
    file_overwrite = False
```

Now generate your static files in your bucket.
```
python manage.py collectstatic
```