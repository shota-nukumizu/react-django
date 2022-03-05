# ReactとDjangoの連携アプリ(React-Django)

# インストール

```
pip install django
pip install djangorestframework
pip install djangocorsheaders
```

以下のコマンドでDjangoプロジェクトを新規作成する

```
django-admin startproject backend
django-admin startapp authentication
```

# セットアップ

## DjangoでJWT認証を実装する

`backend/settings.py`

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # 以下追加
    'rest_framework',
    'rest_framework_simplejwt.token_blacklist',
    'authentication',
]
...

AUTH_USER_MODEL = 'authentication.CustomUser'

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=1),
    'REFRESH_TOKEN_LIFETIME': timedelta(minutes=2),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'VERIFYING_KEY': None,
    'AUTH_HEADER_TYPES': ('JWT',),
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_TYPE_CLAIM': 'token_type',
}
```

# Model作成

`authentication/models.py`

```py
from django.contrib.auth.models import AbstractUser #アカウント名をカスタマイズするにはAbstractUserモジュールを活用する
from django.db import models


class CustomUser(AbstractUser):
    fav_color = models.CharField(blank=True, max_length=120)
```

以下のコマンドを入力する。

```
py manage.py makemigrations
py manage.py migrate
py manage.py createsuperuser
```

# 開発環境

* Python 3.10.1
* Django 4.0.3
* Django REST Framework 3.13
* React
* Visual Studio Code