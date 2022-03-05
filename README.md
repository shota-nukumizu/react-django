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

# 管理サイトへの登録

作成したModelを管理サイトへ登録する。

`authentication/admin.py`

```py
from django.contrib import admin
from .models import CustomUser


class CustomUserAdmin(admin.ModelAdmin):
    model = CustomUser

admin.site.register(CustomUser, CustomUserAdmin)
```

以下のコマンドを入力して、`localhost:5000/admin`にアクセスすれば管理サイトへアクセスできる

```
py manage.py runserver
```

# シリアライザの作成

`authentication/serializers.py`

```py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer


class MyTokenObtainPairSerializer(TokenObtainPairSerializer):

    @classmethod
    def get_token(cls, user):
        token = super(MyTokenObtainPairSerializer, cls).get_token(user)

        token['fav_color'] = user.fav_color
        return token
```

`authentication/urls.py`(Django REST Frameworkのルーティング設定を行う)

```py
from django.urls import path
from rest_framework_simplejwt import views as jwt_views
from .views import ObtainTokenPairWithColorView

# JWT認証を行うためのURLをここで新規作成
urlpatterns = [
    path('token/obtain/', ObtainTokenPairWithColorView.as_view(), name='token_create'),
    path('token/refresh/', jwt_views.TokenRefreshView.as_view(), name='token_refresh'),
]
```

`authentication/views.py`(Viewの設定を行う)

```py
from rest_framework_simplejwt.views import TokenObtainPairView
# 以下のモジュールを追加
from rest_framework import permissions
from .serializers import MyTokenObtainPairSerializer


class ObtainTokenPairWithColorView(TokenObtainPairView):
    permission_classes = (permissions.AllowAny,)
    serializer_class = MyTokenObtainPairSerializer
```

# シリアライザのカスタマイズ

`authentication/serializers.py`

```py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework import serializers
from .models import CustomUser


class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    
    @classmethod
    def get_token(cls, user):
        token = super(MyTokenObtainPairSerializer, cls).get_token(user)

        token['fav_color'] = user.fav_color
        return token

# 以下のプログラムを追加
class CustomUserSerializer(serializers.ModelSerializer):

    email = serializers.EmailField(
        required=True
    )
    username = serializers.CharField()
    password = serializers.CharField(min_length=8, write_only=True)

    class Meta:
        model = CustomUser
        fields = ('email', 'username', 'password')
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        password = validated_data.pop('password', None)
        instance = self.Meta.model(**validated_data)

        if password is not None:
            instance.set_password(password)

        instance.save()
        return instance
```

`authentication/urls.py`

```py
from django.urls import path
from rest_framework_simplejwt import views as jwt_views
from .views import ObtainTokenPairWithColorView, CustomUserCreate #serializers.pyで新規作成したモジュールを追加

urlpatterns = [
    path('user/create/', CustomUserCreate.as_view(), name='create_user'), #新しくルーティングを追加する
    path('token/obtain/', ObtainTokenPairWithColorView.as_view(), name='token_create'),
    path('token/refresh/', jwt_views.TokenRefreshView.as_view(), name='token_refresh'),
]
```

`authentication/views.py`

```py
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework import status, permissions
from rest_framework.response import Response
from rest_framework.views import APIView

from .serializers import MyTokenObtainPairSerializer, CustomUserSerializer


class ObtainTokenPairWithColorView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer

# 新規でアカウントを作成するクラス
class CustomUserCreate(APIView):
    permission_classes = (permissions.AllowAny,)

    def post(self, request, format='json'):
        serializer = CustomUserSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            if user:
                json = serializer.data
                return Response(json, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

# Hello Worldを表示するViewの作成

`authenticaton/views.py`

```py
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework import status, permissions
from rest_framework.response import Response
from rest_framework.views import APIView

from .serializers import MyTokenObtainPairSerializer, CustomUserSerializer


class ObtainTokenPairWithColorView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer


class CustomUserCreate(APIView):
    permission_classes = (permissions.AllowAny,)

    def post(self, request, format='json'):
        serializer = CustomUserSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            if user:
                json = serializer.data
                return Response(json, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

# 新規で作成。HTTPリクエストを介して行われるので、statusを追加の引数でつけておく。
class HelloWorldView(APIView):

    def get(self, request):
        return Response(data={"hello":"world"}, status=status.HTTP_200_OK)
```

`authentication/urls.py`

```py
from django.urls import path
from rest_framework_simplejwt import views as jwt_views
from .views import HelloWorldView, ObtainTokenPairWithColorView, CustomUserCreate

urlpatterns = [
    path('user/create/', CustomUserCreate.as_view(), name='create_user'),
    path('token/obtain/', ObtainTokenPairWithColorView.as_view(), name='token_create'),
    path('token/refresh/', jwt_views.TokenRefreshView.as_view(), name='token_refresh'),
    path('hello/', HelloWorldView.as_view(), name='hello_world'), #新規で追加。ViewはURLのルーティングに都度追加しないと反映されないのでご用心
]
```

# 開発環境

* Python 3.10.1
* Django 4.0.3
* Django REST Framework 3.13
* React
* Visual Studio Code

# 参考サイト

[110% Complete JWT Authentication with Django & React-2020](https://hackernoon.com/110percent-complete-jwt-authentication-with-django-and-react-2020-iejq34ta)