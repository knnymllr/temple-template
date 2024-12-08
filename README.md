# BLUEPRINT

## Confirm installation:

- Python [↗](https://www.python.org/downloads/)
  - `python --version` OR `py, python3`
  - Output: >= 3.11.0
- pip
  - `pip --version`
  - Output: >= 24.0
- Node.js [↗](https://nodejs.org/en/download/package-manager)
  - `node --version`
  - Output: >= 18.15.0
- npm
  - `npm --version`
  - Output: >= 9.5.0
- Docker Desktop [↗](https://docs.docker.com/desktop/install/windows-install/)
  - `docker-compose --version`
  - Output: 2.28.1-desktop.1
- PostgreSQL [↗](https://www.postgresql.org/download/)

## Initial Filetree Setup

- application/
- application/.gitignore
- application/docker-compose.yml
- application/README.md

## Virtual Environment

```
$ cd application
$ python -m venv .venv
$ .venv\Scripts\activate.ps1 (PowerShell)
$ source bin/activate (Linux/MacOS)
$ .venv\Scripts\activate (Windows CMD)

(.venv) C:\...\application

$ python -m pip install --upgrade pip
```

**Add `.venv/` to ./.gitignore. The .venv should never be committed to GitHub, only requirements.txt file. Always check before committing git is not tracking it.**

The virtual environment is only used for local development. It is recommended to avoid installing Python packages globally on your machine, and instead keep track
of specific package versions in a virtual environment. The contents of the virtual environment can be updated in backend/requirements.txt so the Docker container 
knows which packages to install. 

## Create backend project

### Django [↗](https://docs.djangoproject.com/en/5.0/topics/install/)

- `python -m pip install Django`
- `python -m django --version`
- Output >= 5.0.0

```
$ django-admin startproject backend
$ cd backend
$ python manage.py startapp api
$ pip install django-extensions djangorestframework djangorestframework-simplejwt django-cors-headers
$ python -m pip install -U channels["daphne"]
$ pip install channels_redis psycopg2-binary
$ pip freeze > requirements.txt

asgiref==3.8.1
attrs==24.2.0
autobahn==24.4.2
Automat==24.8.1
cffi==1.17.1
channels==4.1.0
channels-redis==4.2.0
constantly==23.10.4
cryptography==43.0.1
daphne==4.1.2
Django==5.1.1
django-cors-headers==4.4.0
django-extensions==3.2.3
djangorestframework==3.15.2
djangorestframework-simplejwt==5.3.1
hyperlink==21.0.0
idna==3.10
incremental==24.7.2
msgpack==1.1.0
psycopg2-binary==2.9.9
pyasn1==0.6.1
pyasn1_modules==0.4.1
pycparser==2.22
PyJWT==2.9.0
pyOpenSSL==24.2.1
redis==5.0.8
service-identity==24.1.0
setuptools==75.1.0
sqlparse==0.5.1
Twisted==24.7.0
txaio==23.1.1
typing_extensions==4.12.2
tzdata==2024.1
zope.interface==7.0.3
```

### Add Files (backend)

- ./.gitignore
- ./Dockerfile
- ./README.md
- ./backend/views.py
- ./api/consumers.py
- ./api/fixtures/
- ./api/routing.py
- ./api/serializers.py
- ./api/urls.py

### Integrate PostgreSQL with Backend

#### backend/urls.py

```py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    # path('api/', include('api.urls')),
]
```

#### backend/settings.py
0. Import at top
`from datetime import timedelta`

1. Below ALLOWED_HOSTS (ln 28)

```py
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    # "https://production-domain.com",
]

CORS_ALLOW_METHODS = [
    'GET',
    'POST',
    'PUT',
    'DELETE',
]

CORS_ALLOW_HEADERS = [
    'Content-Type',
    'Authorization',
]
```

2. Top of Installed Apps (ln 33)

```py
INSTALLED_APPS = [
    'corsheaders',
    'rest_framework',
    'rest_framework_simplejwt',
    'api',
    'django_extensions',
    ...
]
```

3. Top of Middleware (ln 42)

```py
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    ...
]
```

4. After Middleware (ln 51)

```py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

ROOT_URLCONF = 'backend.urls'

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
}
```

5. Update Databases (ln 75)

```py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': 'mypassword',
        'HOST': 'my-postgres',
        'PORT': '5432',
    }
}
```

### Integrate Django Channels

#### backend/asgi.py

```py
import os
from django.core.asgi import get_asgi_application
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.security.websocket import AllowedHostsOriginValidator
from channels.auth import AuthMiddlewareStack
from api.routing import websocket_urlpatterns

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'backend.settings')

django_asgi_app = get_asgi_application()

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AllowedHostsOriginValidator(
        AuthMiddlewareStack(URLRouter(websocket_urlpatterns))
    ),
})
```

#### backend/settings.py

Edit Installed Apps

```py
INSTALLED_APPS = [
    'daphne', # always first
    ...
]

ASGI_APPLICATION = 'backend.asgi.application'

CHANNEL_LAYERS = {
    'default':{
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("redis", 6379)],
        },    
    }
}
```

#### api/consumers.py

```py
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        pass

    async def disconnect(self, close_code): 
        pass

    async def receive(self, text_data):
        pass
```

#### api/routing.py

```py
from django.urls import re_path
from . import consumers

'''
Path will be declared in frontend
let url = `ws://${window.location.host}/ws/socket-server/`;
const chatSocket = new WebSocket(url);
'''
websocket_urlpatterns = [
    re_path(r'ws/socket-server/', consumers.ChatConsumer.as_asgi()) 
]
```

### Additional Sources

- Create a Django project [↗](https://docs.djangoproject.com/en/5.0/intro/tutorial01/#creating-a-project)
- Start a Django app [↗](https://docs.djangoproject.com/en/5.0/intro/tutorial01/#creating-the-polls-app)
- psycopg2 installation [↗](https://www.psycopg.org/install/)
- PostgreSQL and Django [↗](https://stackoverflow.com/questions/5394331/how-to-set-up-a-postgresql-database-in-django)
- Django Extensions [↗](https://django-extensions.readthedocs.io/en/latest/)
- Django Channels [↗](https://channels.readthedocs.io/en/stable/installation.html)
- Redis Channel Layer [↗](https://channels.readthedocs.io/en/stable/topics/channel_layers.html?highlight=redis#redis-channel-layer)
- Channels Redis package [↗](https://pypi.org/project/channels-redis/)
- Redis [↗](https://redis.io/docs/latest/)

## Create frontend project [↗](https://vitejs.dev/guide/#scaffolding-your-first-vite-project)

### Vite

```
$ cd .. (return to application/)
$ npm create vite@latest
? Project name: » frontend
√ Project name: ... frontend
? Select a framework: » - Use arrow-keys. Return to submit.
    Vanilla
    Vue
>   React
    ...
√ Select a framework: » React
? Select a variant: » - Use arrow-keys. Return to submit.
    TypeScript
    TypeScript + SWC
    JavaScript
>   JavaScript + SWC
    Remix ↗
√ Project name: ... frontend
√ Select a framework: » React
√ Select a variant: » JavaScript + SWC

Scaffolding project in C:\Users\...\frontend...
Done. Now run: ...skip, follow instructions below

$ cd frontend
$ npm install
$ npm install @vitejs/plugin-react-swc axios react-router zustand
$ npm install -D vitest 
```

**look into react-konva, @nivo/core, victory**

### Add Files

- ./Dockerfile
- ./src/auth/
- ./src/components/
- ./src/scripts/
- ./src/layouts/
- ./src/pages/
- ./src/stores/
- ./src/styles/

## Integrate Docker

### frontend/

#### Dockerfile

```
FROM node:latest

WORKDIR /frontend

COPY package.json ./
COPY package-lock.json ./

RUN npm install

COPY . .
```

Referenced by application/docker-compose.yml to prepare the backend container with the packages listed in requirements.txt. --host [↗](https://github.com/vitejs/vite/discussions/6146)

#### frontend/vite.config.js

```js
export default defineConfig({
  plugins: [react()],
  server: {
    // override default port 5173
    // @see https://stackoverflow.com/a/74923755
    // @see https://vitejs.dev/config/
    port: 3000,
    host: true,
    strictPort: true,
    watch: {
      usePolling: true,
    },
  },
});
```

#### frontend/package.json

```js
"scripts": {
    "dev": "vite --host",
	...
```

### backend/

#### Dockerfile

```
FROM python:latest

WORKDIR /backend

COPY . .

RUN pip install -r requirements.txt
```

Referenced by application/docker-compose.yml to prepare the backend container with the packages listed in requirements.txt

### application/docker-compose.yml

```
services:
  api:
    build:
      context: ./backend
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/backend
    command: bash -c "python manage.py runserver 0.0.0.0:8000"
    depends_on:
      - my-postgres
  my-postgres:
    image: postgres
    environment:
      POSTGRES_PASSWORD: mypassword
  web:
    build:
      context: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/frontend/src
    command: bash -c "npm run dev"
  redis:
    image: redis:latest
```

## Host the Application

**Make sure Docker Desktop is running**

```
$ cd ..
$ docker-compose up --build
$ docker-compose ps
$ docker network ls
$ docker network inspect <network_name>`

... new terminal window

$ docker-compose exec api bash
$ python manage.py makemigrations
$ python manage.py migrate
$ python manage.py loaddata {json data}
$ python manage.py createsuperuser
```

**Always `docker-compose down` when finished development. Delete containers in Docker Desktop.**
