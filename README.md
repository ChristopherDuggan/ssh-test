 Let's start by setting up a basic Django project that includes Django Rest Framework with the usual steps:

1. `mkvirtualenv social-auth`
2.  `pip install django` (and don't forget our `pip freeze > requirements.txt` after each installation)
3. `django-admin startproject social_auth .`
4. create the PostgreSQL database
```sql
CREATE ROLE social_auth_admin WITH PASSWORD '12345';

CREATE DATABASE social_auth WITH OWNER social_auth_admin;

GRANT ALL PRIVILEGES ON DATABASE social_auth TO social_auth_admin;
```
5. `pip install psycopg2-binary`
6. Alter our `settings.py` to use our `psql` database
```py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'social_auth',
        'USER': 'social_auth_admin',
        'PASSWORD': '12345',
        'HOST': 'localhost'
    }
}
```
7. Run our migrations with `python manage.py makemigrations && python manage.py migrate`

Now that all the standard steps are out of the way, lets make a rudimentary social media app.

8. `django-admin startapp social_auth_app`
9. Now lets add it to our `INSTALLED_APPS` in `settings.py`

```py
INSTALLED_APPS = [
    ...
    'social_auth_app',
]
```

10. Now lets add the following to our `social_auth_app/models.py`

```py

from django.db import models
from django.contrib.auth.models import User

# Create your models here.

class User_profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, default=4)
    name = models.CharField(max_length=50)
    email = models.CharField(max_length=50)
    def __str__(self):
        return self.name

class Post(models.Model):
    user = models.ForeignKey(User_profile, on_delete=models.CASCADE)
    content = models.CharField(max_length=140)

    def __str__(self):
        return self.content

class Comment(models.Model):
    user = models.ForeignKey(User_profile, on_delete=models.CASCADE)
    content = models.CharField(max_length=140)
    post= models.ForeignKey(Post, on_delete=models.CASCADE)

    def __str__(self):
        return self.content
```
We're making a model for our user profiles that has a [one-to-one](https://docs.djangoproject.com/en/4.1/topics/db/examples/one_to_one/)
relationship with the default Django `User` model. Why? It allows us to
make a model that maps precisely to our users but can be customized to be
more robust.

Our `Post` model relates to our `User_profile` and our `Comment` model
relates to a `User` and to a `Post`.

Don't forget to migrate!

11. Now lets register our models in `social_auth_app/admin.py`

```py
from django.contrib import admin
from .models import User_profile, Post, Comment

# Register your models here.

admin.site.register(User_profile)
admin.site.register(Post)
admin.site.register(Comment)
```

12. Run `python manage.py createsuperuser` to add an admin account to our
application. Once logged in, we can check and see if all the fileds are
working by creating a user profile, post, and comment.

INSERT PNGS FROM HOME/PICTURES/SCREENSHOTS

13. Now we're ready to install Django Rest Framework with `pip install
djangorestframework` (and update requirements.txt) and update `settings.py`

```py
INSTALLED_APPS = [
    ...
    'social_auth_app',
    'rest_framework',
]
```

14. Next, let's make a `social_auth_app/serializers.py` that looks like this

```py
from rest_framework import serializers
from .models import User_profile, Comment, Post
class User_profileSerializer(serializers.ModelSerializer):
    class Meta:
        model = User_profile
        fields = '__all__'

class Comment_Serializer(serializers.ModelSerializer):
  class Meta:
      model = Comment
      fields = '__all__'
      
class Post_Serializer(serializers.ModelSerializer):
  class Meta:
    model = Post
    fields = '__all__'
    
class Post_Serializer(serializers.BaseSerializer):
  def to_representation(self,instance):
    return {
      "id":instance.id,
      "content": instance.content,
      "user": instance.user.name,
    }
    
class Comment_Serializer(serializers.BaseSerializer):
    def to_representation(self,instance):
      return {
      "id":instance.id,
      "content": instance.content,
      "user": instance.user.name,
    }
```

The `Post` and `Comment` serializers have both a `serializers.ModelSerializer`
definition and a `serializers.BaseSerializer` definition. The
[ModelSerializer](https://www.django-rest-framework.org/api-guide/serializers/#modelserializer)
is what we typically use when we want some standard CRUD functionality where the
[BaseSerializer](https://www.django-rest-framework.org/api-guide/serializers/#read-only-baseserializer-classes)
is set up to provide read-only information. We'll see why we want both a little
later on.

15. Now we can build out our our `social_auth_app/views.py`:

```py
from rest_framework import viewsets
from .models import User_profile, Comment, Post
from rest_framework.views import APIView
from .serializers import User_profileSerializer, Comment_Serializer, Post_Serializer, Post_Serializer
from rest_framework import permissions
from rest_framework.response import Response

# Create your views here.

class UserProfile_ViewSet(viewsets.ModelViewSet):
  queryset = User_profile.objects.all()
  serializer_class = User_profileSerializer;

class Comment_ViewSet(viewsets.ModelViewSet):
  queryset = Comment.objects.all()
  serializer_class = Comment_Serializer;

class Post_ViewSet(viewsets.ModelViewSet):
  queryset = Post.objects.all()
  serializer_class = Post_Serializer;

class AllPost_ViewSet(APIView):
  permission_classes = [
    permissions.AllowAny
  ]
  def post(self,request):
    try:
      user = self.request.user
      isAuthenticated = user.is_authenticated
      if isAuthenticated:
          content = request.data['content']
          userProfile = User_profile.objects.get(user=user)
          Post.objects.create(user=userProfile, content=content)
          return Response({'message': "Post Successfully Created!"})
      else:
          return Response({'error': "not authenticated make sure you include a token"})
    except:
          return Response({'error': "error; you are most likely messed up by passing an invaild body"})
  def get(self, request):
    try:
      results = Post.objects.all()
      all_post = Post_Serializer(results, many=True)
      return Response(all_post.data)
    except:
      return Response({"error": "something went wrong"})

class OnePost_ViewSet(APIView):
  permission_classes = [
    permissions.AllowAny
  ]
  def get(self, request,id):
    try:
      post_results = Post.objects.get(id=id)
      post = Post_Serializer(post_results)
      comments_results = Comment.objects.filter(post=id)
      comments = Comment_Serializer(comments_results, many=True)
      return Response({"post":post.data,"comments":comments.data})
    except:
      return Response({"error": "something went wrong"})

class Comment_ViewSet(APIView):
  permission_classes = [
    permissions.AllowAny
  ]
  def post(self,request,id):
    try:
      user = self.request.user
      isAuthenticated = user.is_authenticated
      if isAuthenticated:
          content = request.data['content']
          userProfile = User_profile.objects.get(user=user)
          post = Post.objects.get(id=id)
          Comment.objects.create(user=userProfile, content=content,post=post)
          return Response({'message': "Comment Successfully Created!"})
      else:
          return Response({'error': "not authenticated make sure you include a token"})
    except:
          return Response({'error': "error; you are most likely messed up by passing an invaild body"})
```

There's not much you haven't seen here until we get down to the
`Post_ViewSet`. There we start to see 
```py
    permission classes = [
        permissions.AllowAny
    ]
```
[Permissions](https://www.django-rest-framework.org/api-guide/permissions/) are
used to grant or deny access for different classes of users to different parts
of the
API. [`AllowAny`](https://www.django-rest-framework.org/api-guide/permissions/#allowany)
allows unrestricted access, regardless of if the request was authenticated or unauthenticated.

```py
    def post(self,request):
        try:
            user = self.request.user
            isAuthenticated = user.is_authenticated
            if isAuthenticated:
                content = request.data['content']
                userProfile = User_profile.objects.get(user=user)
                Post.objects.create(user=userProfile, content=content)
                return Response({'message': "Post Successfully Created!"})
```
This bit checks to see if the user is authenticated and, if so, has that user
create the post.

The `else` triggers if the user is not authenticated and the `except`
triggers if something goes wrong with the `try`. It's the equivalent of
the `catch` in the try-catch setup in JS

16. With the views configured, we can move on to
`social_auth_app/urls.py`:
```py
from django.urls import path
from .views import AllPost_ViewSet,OnePost_ViewSet,Comment_ViewSet
urlpatterns = [
  path('posts', AllPost_ViewSet.as_view()),
  path('posts/<int:id>', OnePost_ViewSet.as_view()),
  path('posts/<int:id>/comments', Comment_ViewSet.as_view()),
]
```

17. Of course now, we have to update the `urls.py` in our "core" app,
`social_auth/urls.py`:
```py
from django.contrib import admin
from django.urls import path, include
from rest_framework import routers
from social_auth_app.views import UserProfile_ViewSet, Post_ViewSet

router = routers.DefaultRouter()

router.register(r'profile', UserProfile_ViewSet, basename='profile')
router.register(r'post', Post_ViewSet, basename='post')

urlpatterns = [
    path('', include(router.urls)),
    path('', include('social_auth_app.urls')),
    path('admin/', admin.site.urls),
]
```

Ok cool. That's all well and good but what about the user authentication?
Let's set up a brand new app to handle all that.

18. `django-admin startapp user_auth`
19. Update`INSTALLED_APPS` in `settings.py`

```py
INSTALLED_APPS = [
    ...
    'social_auth_app',
    'rest_framework',
    'user_auth',
]
```
Unlike with `social_auth_app` we're only going to need to edit our
`serializer.py`, `views.py`, and `urls.py`. 

20. First, `serializers.py`:

```py
from rest_framework import serializers

class UserSerializer(serializers.BaseSerializer):
  def to_representation(self,instance):
    return {
      "id":instance.id,
      "username": instance.username,
      "password": instance.password,
    }
```

21. Then `views.py` (it won't work yet):
```py
from rest_framework.views import APIView
from django.contrib.auth.models import User
from django.contrib import auth
from rest_framework.response import Response
from .serializers import UserSerializer
from rest_framework import permissions
from knox.models import AuthToken
from social_auth_app.models import User_profile
from social_auth_app.serializers import User_profileSerializer

# Create your views here.

class SignupView(APIView):
  permission_classes = [
    permissions.AllowAny
  ]

  def get(self,request):
      result = User.objects.all()
    all_users = UserSerializer(result,many=True)
    return Response(all_users.data)
    def post(self,request):
        # pulls all of the user data from the request
        data = self.request.data
        username = data["username"]
        email = data["email"]
        password = data["password"]
        re_password = data["re_password"]
        try:
            # check and see if the password matches the verification    
            if password == re_password:
                #check and see if the username already exists in th e database
                if User.objects.filter(username=username).exists():
                    return Response({"error": "Username already exists"})
                else:
                    # if the user doesn't already exist, create them using the data from the post request
                    user = User.objects.create_user(
                        username=username, password=password)
                    User_profile.objects.create(user=user,email=email,name=username)
                    return Response({
                        "success": "User created successfully",
                        # creates an authtoken for the user
                        "token": AuthToken.objects.create(user)[1]
                    })
            else:
                return Response({"error": "Passwords do not match"})
        except:
            return Response({"error": "Something went wrong signing up"})

class LoginView(APIView):
    permission_classes = [
        permissions.AllowAny
    ]

    def post(self,request):
        data = self.request.data
        username = data["username"]
        password = data["password"]
        try:
            # checks if username and password are correct
            user = auth.authenticate(username=username, password=password)

            # if the user exists, tries to login
            if user is not None:
                auth.login(request, user)
                return Response({"success": "User authenticated",
                                 # creates an authtoken for the user
                                 "token": AuthToken.objects.create(user)[1]})
            else:
                return Response({"error": "Error Authenticating"})
        except:
            return Response({"error": "Something went wrong when logging in"})

class GrabProfile(APIView):
    def get(self, request):
        try:
            user = self.request.user
            # grabs the profile from the database that matches the requested user
            profile = User_profile.objects.get(user=user)

            # converts the profile to JSON
            profile_json = User_profileSerializer(profile)
            return Response({"profile": profile_json.data})
        except:
            return Response({"error": "no user profile found"})
```

You may have noticed an import toward the top you haven't seen before:

```py
...
from knox.models import AuthToken
...
```
[Knox](https://james1345.github.io/django-rest-knox/) is a relatively
painless way to provide token-based authentication for Django REST Framework

Lets install it!

22. Run `pip install django-rest-knox` then `pip freeze > requirements.txt`
23. Add `'knox'` to the `INSTALLED_APPS`
```py
INSTALLED_APPS[
    ...
    'social_auth_app',
    'rest_framework',
    'user_auth',
    'knox',
]
```
24. And add this to the bottom of `settings.py`:
```py
from datetime import timedelta
REST_KNOX = {
  'TOKEN_TTL': timedelta(hours=340),
  'AUTO_REFRESH': True,
}

REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'knox.auth.TokenAuthentication',
    ]
}
```
The information inside of `REST_KNOX` configures Knox's setup.
[`TOKEN_TTL`](https://james1345.github.io/django-rest-knox/settings/#token_ttl)
sets how long a token exists before expiring.

In the `REST_FRAMEWORK` configuration, setting the 
[`DEFAULT_PERMISSION_CLASSES`](https://www.django-rest-framework.org/api-guide/permissions/#setting-the-permission-policy)
to
[`IsAuthenticatedOrReadOnly`](https://www.django-rest-framework.org/api-guide/permissions/#isauthenticatedorreadonly)
lets authenticated users perform full CRUD while safely allowing
everyone else to just read the data.

Setting the
[`DEFAULT_AUTHENTICATION_CLASSES`](https://www.django-rest-framework.org/api-guide/authentication/#setting-the-authentication-scheme)
to
[`knox.auth.TokenAuthentication`](https://james1345.github.io/django-rest-knox/auth/#tokenauthentication)
makes Rest Framework use Knox instead of its default.

25. Lets run a quick migration (`python manage.py makemigrations && python
manage.py migrate`). 

26. Now lets set up `user_auth/urls.py`:
# ssh-test
