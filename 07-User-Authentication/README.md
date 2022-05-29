<div dir="rlt">
  
# احراز هویت کاربر
  
  
در فصل قبلی ما دسترسی ای پی آی های مان را آپدیت کردیم ، که به ان مجوز گویند . در این فصل  ، ما احراز هویت را پیاده سازی  می‌کنیم فرایندی که کاربر  می‌تواند یک حساب جدید ایجاد کند ، به آن حساب وارد شود و از آن حساب خارج شود .  احراز هویت در یک وب سایت جنگو یکپارچه ساده است و متاثر از یک الگوی کوکی مبتنی بر جلسه است ، که پایین تر آن را مورد بررسی قرار خواهیم داد .اما با وجود یک ای پی آی ، کارها کمی سخت تر می شود . به یاد داشته باشید که Http یک  پروتکل بدون حفظ حالت  است پس هیچ راهی برای به خاطر سپاری وجود ندارد که یک کاربر از یک درخواست به درخواست بعدی احراز هویت شده یا خیر .هر بار که یک  کاربر درخواست یک منبع محدود شده را میدهد در نتیجه باید آن مورد تایید واقع شود. 
  
راه حل ، ارسال یک نشان یکتا همراه هر درخواست می باشد   .  به صورت گیج کننده ای ،یک دیدگاه  توافق شده جهانی برای این نشان یکتا تعریف نشده است  و آن می تواند چندین فرم داشته باشد .  جنگو رست با [چهار نوع آپشن مختلف احراز هویت شکل دهی می شود](https://www.django-rest-framework.org/api-guide/authentication/#api-reference) :  پایه ، جلسه ، تو که ، و پیش فرض .پکیج های ثانویه زیادی وجود دارند که  فیچر های  اضافه مانند  جیسون وب تو کن  ارائه می دهند.
  در این فصل   بررسی می‌کنیم که یکی پی آی احراز هویت چگونه کار میکند ، و سپس یک انتخاب آگاهانه برای وبلاگ خود انجام می دهیم و ای پی آی هایی برای ایجاد حساب ، ورود به حساب و  خروج از حساب پیاده سازی می کنیم 

### احراز هویت پایه

رایج ترین فرم احراز هویت Http به عنوان [احرازهویت ”پایه“](https://datatracker.ietf.org/doc/html/rfc7617) شناخته می‌شود. زمانی که یک کاربر درخواست http ارسال می کند،  قبل از اعطای دسترسی ،  مجبور به ارسال یک  گواهی احراز هویت تایید شده است.  
  
پروسه کامل درخواست/ پاسخ به صورت زیر است
  

-  1 . کاربر یک درخواست http ارسال می کند
-  2 . سرور یک پاسخ با استاتوس ۴۰۱ و یک هدر  WWW-Authenticate HTTP با جزئیات چگونگی دسترسی ارسال می کند  
-  3 . کاربر گواهی خود را از طریق یک هدر [Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) HTTP ارسال می کند    
-  4 . سرور گواهی را چک میکند و پاسخ را همراه کد ۲۰۰ یا ۴۰۳ به سمت کاربر ارسال می کند یک بار که کاربر تایید شد ، می تواند تمام درخواست های بعدی خود را با هدر Authorization HTTP  ارسال کند.  

  
Once approved, the client sends all future requests with the Authorization HTTP header
credentials. We can also visualize this exchange as follows:
  
  
  
  
<div dir="ltr">
  
Diagram
```code
Client                                                                                                                  Server
------                                                                                                                  ------
  
--------------------------------------->
GET / HTTP/1.1
  
                                                                                        <-------------------------------------
                                                                                                     HTTP/1.1 401 Unauthorized
                                                                                                     WWW-Authenticate: Basic
                                                                                                                               
                                                                                                                               
--------------------------------------->
GET / HTTP/1.1
Authorization: Basic d3N2OnBhc3N3b3JkMTIz
                                                                                          
                                                                                          
                                                                                         <-------------------------------------
                                                                                                                HTTP/1.1 200 OK
```
  
</div>  
  
Note that the authorization credentials sent are the unencrypted [base64 encoded](https://en.wikipedia.org/wiki/Base64) version of
<username>:<password>. So in my case, this is wsv:password123 which with base64 encoding is
d3N2OnBhc3N3b3JkMTIz.  
                                                                                          
  
  
The primary advantage of this approach is its simplicity. But there are several major downsides.
First, on every single request the server must look up and verify the username and password,
which is inefficient. It would be better to do the look up once and then pass a token of some
kind that says, this user is approved. Second, user credentials are being passed in clear text—not
encrypted at all—over the internet. This is incredibly insecure. Any internet traffic that is not
encrypted can easily be captured and reused. Thus basic authentication should **only** be used via
[HTTPS](https://en.wikipedia.org/wiki/HTTPS), the secure version of HTTP.

  
### Session Authentication
  

Monolithic websites, like traditional Django, have long used an alternative authentication scheme
that is a combination of sessions and cookies. At a high level, the client authenticates with its
credentials (username/password) and then receives a session ID from the server which is stored
as a cookie. This session ID is then passed in the header of every future HTTP request.
When the session ID is passed, the server uses it to look up a session object containing all
available information for a given user, including credentials.
  
This approach is **stateful** because a record must be kept and maintained on both the server (the
session object) and the client (the session ID).
  
Let’s review the basic flow:
  
  
- 1 A user enters their log in credentials (typically username/password)
- 2 The server verifies the credentials are correct and generates a session object that is then
stored in the database
- 3 The server sends the client a session ID—not the session object itself—which is stored as a
cookie on the browser
- 4 On all future requests the session ID is included as an HTTP header and, if verified by the
database, the request proceeds
- 5 Once a user logs out of an application, the session ID is destroyed by both the client and
server
- 6 If the user later logs in again, a new session ID is generated and stored as a cookie on the
client

  
  
The default setting in Django REST Framework is actually a combination of Basic Authentication
and Session Authentication. Django’s traditional session-based authentication system is used
and the session ID is passed in the HTTP header on each request via Basic Authentication.
  
  
The advantage of this approach is that it is more secure since user credentials are only sent once,
not on every request/response cycle as in Basic Authentication. It is also more efficient since
the server does not have to verify the user’s credentials each time, it just matches the session ID
to the session object which is a fast look up.
  
  
There are several downsides however. First, a session ID is only valid within the browser where
log in was performed; it will not work across multiple domains. This is an obvious problem when
an API needs to support multiple front-ends such as a website and a mobile app. Second, the
session object must be kept up-to-date which can be challenging in large sites with multiple
servers. How do you maintain the accuracy of a session object across each server? And third, the
cookie is sent out for every single request, even those that don’t require authentication, which
is inefficient.
  
  
As a result, it is generally not advised to use a session-based authentication scheme for any API
that will have multiple front-ends.

 
### Token Authentication
  
  
The third major approach–and the one we will implement in our Blog API–is to use token
authentication. This is the most popular approach in recent years due to the rise of single page
applications.  

  
Token-based authentication is **stateless**: once a client sends the initial user credentials to the
server, a unique token is generated and then stored by the 
client as either a cookie or in [local storage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage). 
This token is then passed in the header of each incoming HTTP request and the server
uses it to verify that a user is authenticated. The server itself does not keep a record of the user,
just whether a token is valid or not.
 


**Cookies vs localStorage**
  
Cookies are used for reading **server-side** information. They are smaller (4KB) in size and automatically sent with each HTTP request.
LocalStorage is designed for **client-side** information.
It is much larger (5120KB) and its contents are not sent by default with each HTTP request.
Tokens stored in both cookies and localStorage are vulnerable to XSS attacks. The current
best practice is to store tokens in a cookie with the httpOnly and Secure cookie flags.

  
Let’s look at a simple version of actual HTTP messages in this challenge/response flow. Note that
the HTTP header WWW-Authenticate specifies the use of a Token which is used in the response
Authorization header request.

  
  
<div dir="ltr">
  
Diagram
```code
Client                                                                                                                  Server
------                                                                                                                  ------
  
--------------------------------------->
GET / HTTP/1.1
  
                                                                                        <-------------------------------------
                                                                                                     HTTP/1.1 401 Unauthorized
                                                                                                     WWW-Authenticate: Token
                                                                                                                               
                                                                                                                               
--------------------------------------->
GET / HTTP/1.1
Authorization: Token 401f7ac837da42b97f613d789819ff93537bee6a
                                                                                          
                                                                                          
                                                                                         <-------------------------------------
                                                                                                                HTTP/1.1 200 OK
```
  
</div>  
  
 
                                                                                         
There are multiple benefits to this approach. Since tokens are stored on the client, scaling the
servers to maintain up-to-date session objects is no longer an issue. And tokens can be shared
amongst multiple front-ends: the same token can represent a user on the website and the same
user on a mobile app. The same session ID can not be shared amongst different front-ends, a
major limitation.
                                                                                          
A potential downside is that tokens can grow quite large. A token contains all user information,
not just an id as with a session id/session object set up. Since the token is sent on every request,
managing its size can become a performance issue.                                                                                         
                                                                                         
   
Exactly how the token is implemented can also vary substantially. Django REST Frameworks’
built-in [TokenAuthentication](http://www.django-rest-framework.org/api-guide/authentication/#tokenauthentication) 
is deliberately quite basic. As a result, it does not support setting
tokens to expire, which is a security improvement that can be added. It also only generates one
token per user, so a user on a website and then later a mobile app will use the same token.
Since information about the user is stored locally, this can cause problems with maintaining and
updating two sets of client information.
                                                                                          
                                                                                          
JSON Web Tokens (JWTs) are a new, enhanced version of tokens that can be added to Django
REST Framework via several third-party packages. JWTs have several benefits including the
ability to generate unique client tokens and token expiration. They can either be generated on
the server or with a third-party service like [Auth0](https://auth0.com/). And JWTs can be encrypted which makes
them safer to send over unsecured HTTP connections.
                                                                                          
 
Ultimately the safest bet for most web APIs is to use a token-based authentication scheme. JWTs
are a nice, modern addition though they require additional configuration. As a result, in this book
we will use the built-in `TokenAuthentication`.
  
 
### Default Authentication

  
 
The first step is to configure our new authentication settings. Django REST Framework comes
with a [number of settings](https://www.django-rest-framework.org/api-guide/settings/) that are implicitly set. For example, `DEFAULT_PERMISSION_CLASSES`
was set to AllowAny before we updated it to `IsAuthenticated`.
                                                                                          
The `DEFAULT_AUTHENTICATION_CLASSES` are set by default so let’s explicitly add both SessionAuthentication
and BasicAuthentication to our `config/settings.py` file.
                                                                                          
                                                                                          
                                                                                          
<div dir="ltr">
  
Code
```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [ # new
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication'
    ],
}
```
  
</div>                                                                                         
                                                                                          
                                                                                          
Why use **both** methods? The answer is they serve different purposes. Sessions are used to power
the Browsable API and the ability to log in and log out of it. BasicAuthentication is used to pass
the session ID in the HTTP headers for the API itself.
                                                                                          
If you revisit the browsable API at http://127.0.0.1:8000/api/v1/ it will work just as before.
Technically, nothing has changed, we’ve just made the default settings explicit.                                                                                        
                                                                                          
                                                                                          
### Implementing token authentication                                                                                    
                                                                                          

Now we need to update our authentication system to use tokens. The first step is to update our
`DEFAULT_AUTHENTICATION_CLASSES` setting to use TokenAuthentication as follows:
                                                                                          
  
<div dir="ltr">
  
Code
```python
# config/settings.py
REST_FRAMEWORK = {
'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication', # new
    ],
}

```
  
</div> 
                                                                                          
  
We keep `SessionAuthentication` since we still need it for our Browsable API, but now use tokens
to pass authentication credentials back and forth in our HTTP headers. We also need to add the
authtoken app which generates the tokens on the server. It comes included with Django REST
Framework but must be added to our INSTALLED_APPS setting:

                                                                                          
<div dir="ltr">
  
Code
```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
  
    # 3rd-party apps
    'rest_framework',
    'rest_framework.authtoken', # new
  
    # Local
    'posts',
]
```
  
</div>                                                                                          
                                                                                          
Since we have made changes to our `INSTALLED_APPS` we need to sync our database. Stop the
server with `Control+c`. Then run the following command.                                                                                         
                                                                                          

<div dir="ltr">
  
Command Line
```shell
(blogapi) $ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, authtoken, contenttypes, posts, sessions
Running migrations:
  Applying authtoken.0001_initial... OK
  Applying authtoken.0002_auto_20160226_1747... OK
```
  
</div> 
     
                                                                                          
Now start up the server again .                                                                                     
                                                                                          
   
<div dir="ltr">
  
Command Line
```shell
(blogapi) $ python manage.py runserver
```
  
</div> 
                                                                                          
                                                                                          
If you navigate to the Django admin at http://127.0.0.1:8000/admin/ you’ll see there is now
a Tokens section at the top. Make sure you’re logged in with your superuser account to have
access.                                                                                          
   
![Admin Homepage with Tokens](images/1.jpg)
  
Click on the link for Tokens. Currently there are no tokens which might be surprising.

                                                                                          
![Admin Tokens Page](images/2.jpg)                                                                                          
                                    
                                                                                          
After all we have existing users. However, the tokens are only generated after there is an API call
for a user to log in. We have not done that yet so there are no tokens. We will shortly.
                                                                                          
  
### Endpoints                                                                                        
                                                                                          
                                                                                          
We also need to create endpoints so users can log in and log out. We could create a dedicated
users app for this purpose and then add our own urls, views, and serializers. However user
authentication is an area where we really do not want to make a mistake. And since almost all APIs
require this functionality, it makes sense that there are several excellent and tested third-party
packages we can use it instead.
                                                                                          
Notably we will use [dj-rest-auth](https://github.com/jazzband/dj-rest-auth) in combination with [django-allauth](https://github.com/pennersr/django-allauth)
to simplify things. Don’t feel bad about using third-party packages. They exist for a reason and even the best Django
professionals rely on them all the time. There is no point in reinventing the wheel if you don’t
have to!
                                                                                          
                                                                                          
### dj-rest-auth                                                                                        
                                                                                          
First we will add log in, log out, and password reset API endpoints. These come out-of-the-box
with the popular `dj-rest-auth package`. Stop the server with `Control+c` and then install it.                                                                                       
                                                                                          
                                                                                          
<div dir="ltr">
  
Command Line
```shell
(blogapi) $ pipenv install dj-rest-auth==1.1.0
```
  
</div>                                                                                         
                                                                                          
    
Add the new app to the INSTALLED_APPS config in our `config/settings.py` file.                                                                                          

                                                                                          
<div dir="ltr">
  
Code
```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
  
    # 3rd-party apps
    'rest_framework',
    'rest_framework.authtoken',
    'dj_rest_auth', # new
  
    # Local
    'posts',
]

```
  
</div>                       

                                                                                          
                                                                                          
Update our `config/urls.py` file with the `dj_rest_auth` package. We’re setting the URL routes
to api/v1/dj-rest-auth. Make sure to note that URLs should have a dash - not an underscore
_, which is an easy mistake to make.
                                                                                          
                                                                                          
<div dir="ltr">
  
Code
```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path
  
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('posts.urls')),
    path('api-auth/', include('rest_framework.urls')),
    path('api/v1/dj-rest-auth/', include('dj_rest_auth.urls')), # new
]
```
  
</div>                                                                                       
                                                                                          
                                                                                          
And we’re done! If you have ever tried to implement your own user authentication endpoints, it
is truly amazing how much time—and headache—dj-rest-auth saves for us. Now we can spin up
the server to see what dj-rest-auth has provided.                                                                                      
                                                                                          

<div dir="ltr">
  
Command Line
```shell
(blogapi) $ python manage.py runserver
```
  
</div>
                                                                                          

We have a working log in endpoint at http://127.0.0.1:8000/api/v1/dj-rest-auth/login/.
                                                                                          
                                                                                          
![API Log In Endpoint](images/3.jpg)                                                                                          
             
                                                                                          
And a log out endpoint at http://127.0.0.1:8000/api/v1/dj-rest-auth/logout/.                                                                                         
              
                                                                                          
                                                                                          
![API Log Out Endpoint](images/4.jpg)                                                                                      
                                                                                          
There are also endpoints for password reset, which is located at:

                                                                                          
http://127.0.0.1:8000/api/v1/dj-rest-auth/password/reset                                                                               
   

![API Password Reset](images/5.jpg)
                                                                                          
                                                                                          
                                                                                          
And for password reset confirmed:
                                                                                          
http://127.0.0.1:8000/api/v1/dj-rest-auth/password/reset/confirm                                                                                         
                                                                                          
                                                                                          
![API Password Reset Confirm](images/6.jpg)                                                                                      
                                                                                          
 
### User Registration
                                                                                         
                                                                                          
Next up is our user registration, or sign up, endpoint. Traditional Django does not ship with builtin views or 
URLs for user registration and neither does Django REST Framework. Which means
we need to write our own code from scratch; a somewhat risky approach given the seriousness–
and security implications–of getting this wrong                                                                                          
                                                                                          
                                                                                          
A popular approach is to use the third-party package [django-allauth](https://github.com/pennersr/django-allauth) which comes with user
registration as well as a number of additional features to the Django auth system such as social authentication via Facebook, Google, Twitter, etc.
If we add dj_rest_auth.registration from the dj-rest-auth package then we have user registration endpoints too!

                                                                                        
Stop the local server with `Control+c` and install django-allauth.                                                                                         
                                                                                          
 
<div dir="ltr">
  
Command Line
```shell
(blogapi) $ pipenv install django-allauth~=0.42.0
```
  
</div>
                                                                                          
                                                                                          
Then update our `INSTALLED_APPS` setting. We must add several new configs:                                                                                        
                                                                                          
                                                                                          
- django.contrib.sites
- allauth
- allauth.account
- allauth.socialaccount
- dj_rest_auth.registration                                                                                          
                                                                                          
   
Make sure to also include EMAIL_BACKEND and SITE_ID. Technically it does not matter where in
the `config/settings.py` file they are placed, but it’s common to add additional configs like that
at the bottom.

                                                                                          
                                                                                          
<div dir="ltr">
  
Code
```python
# config/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites', # new
  
    # 3rd-party apps
    'rest_framework',
    'rest_framework.authtoken',
    'allauth', # new
    'allauth.account', # new
    'allauth.socialaccount', # new
    'dj_rest_auth',
    'dj_rest_auth.registration', # new
  
    # Local
    'posts',
]
  
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend' # new
  
SITE_ID = 1 # new
```
  
</div>                                                                                          
                                                                                          
                                                                                        
The email back-end config is needed since by default an email will be sent when a new user is
registered, asking them to confirm their account. Rather than also set up an email server, we will
output the emails to the console with the console.EmailBackend setting.
                                                                                          
                                                                                          
`SITE_ID` is part of the built-in Django [“sites” framework](https://docs.djangoproject.com/en/3.1/ref/contrib/sites/), which is a way to host multiple
websites from the same Django project. We obviously only have one site we are working on here
but django-allauth uses the sites framework, so we must specify a default setting.
                                                                                          
                                                                                          
Ok. We’ve added new apps so it’s time to update the database.                                                                                          
                                                                                          
                                                                                          
<div dir="ltr">
  
Command Line
```shell
(blogapi) $ python manage.py migrate
```
  
</div>                                                                                          
                                                                                          
                                                                                          
Then add a new URL route for registration.                                                                                       
                                                                                          
                                                                                          
<div dir="ltr">
  
Code
```python
# config/urls.py
from django.contrib import admin
from django.urls import include, path
  
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('posts.urls')),
    path('api-auth/', include('rest_framework.urls')),
    path('api/v1/dj-rest-auth/', include('dj_rest_auth.urls')),
    path('api/v1/dj-rest-auth/registration/', # new
    include('dj_rest_auth.registration.urls')),
]
```
  
</div>                                                                                         
                                                                                          
                                                                                          
And we’re done. We can run the local server.                                                                                          
                                                                                          
                                                                                          
<div dir="ltr">
  
Command Line
```shell
(blogapi) $ python manage.py runserver
```
  
</div>                                                                                          
                                                                                          
                                                                                          
                                                                                          
There is now a user registration endpoint at http://127.0.0.1:8000/api/v1/dj-rest-auth/registration/.
                                                                                          
                                                                                          
                                                                                          
                                                                                          
![API Register](images/7.jpg)                                                                                         
                                                                                          
                                                                                          
### Tokens                                                                                        
      
                                                                                          
To make sure everything works, create a third user account via the browsable API endpoint. I’ve
called my user testuser2. Then click on the “POST” button.                                                                                        
                                                                                          
                                                                                          
![API Register New User](images/8.jpg)                                                                                        
                                                                                          
The next screen shows the HTTP response from the server. Our user registration POST was
successful, hence the status code HTTP 201 Created at the top. The return value key is the
auth token for this new user.                                                                                         
                                                                                          
                                                                                          
![API Auth Key](images/9.jpg)                                                                                         
                                                                                          
                                                                                          
If you look at the command line console, an email has been automatically generated by django-allauth.
This default text can be updated and an email SMTP server added with additional configuration
that is covered in the book [Django for Beginners](https://djangoforbeginners.com/).                                                                                         
                                                                                          
   
<div dir="ltr">
  
Command Line
```shell
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Subject: [example.com] Please Confirm Your E-mail Address
From: webmaster@localhost
To: testuser2@email.com
Date: Wed, 29 Jul 2020 20:54:26 -0000
Message-ID:
  <159605606600.8206.5520712009851546888@1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0\
.0.0.0.0.0.0.0.ip6.arpa>
    
Hello from example.com!
    
You're receiving this e-mail because user testuser2 has given yours as an e-mail address \
to connect their account.
    
To confirm this is correct, go to http://127.0.0.1:8000/api/v1/dj-rest-auth/\
registration/account-confirm-email/MQ:1k0t5m:6l0l09er1p_cbxgkJWDuSw2j00M/
    
Thank you from example.com!
example.com
```
  
</div>                                                                                         
                                                                                          
                                                                                          
Switch over to the Django admin in your web browser at http://127.0.0.1:8000/admin/. You
will need to use your superuser account for this. Then click on the link for Tokens at the top of
the page. You will be redirected to the Tokens page.
                                                                                        
                                                                                          
                                                                                          
![Admin Tokens](images/10.jpg)                                                                                        
                                                                                          
                                                                                          
A single token has been generated by Django REST Framework for the testuser2 user. As
additional users are created via the API, their tokens will appear here, too.
  
  
A logical question is, Why are there are no tokens for our superuser account or testuser? The
answer is that we created those accounts before token authentication was added. But no worries,
once we log in with either account via the API a token will automatically be added and available.
  
  
Moving on, let’s log in with our new testuser2 account. In your web browser, navigate to
http://127.0.0.1:8000/api/v1/dj-rest-auth/login/. Enter the information for our testuser2
account. Click on the “POST” button.                                                                                         
                                                                                          
                                                                                          
                                                                                          
![API Log In testuser2](images/11.jpg)                                                                                          
         
  
Two things have happened. In the upper righthand corner, our user account testuser2 is visible,
confirming that we are now logged in. Also the server has sent back an HTTP response with the
token.                                                                                       

  
![API Log In Token](images/12.jpg)
  
  
In our front-end framework, we would need to capture and store this token. Traditionally this
happens on the client, either in [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) or as a cookie, and then all future requests include
the token in the header as a way to authenticate the user. Note that there are additional security
concerns on this topic so you should take care to implement the best practices of your front-end
framework of choice.

  
### Conclusion  
  
  
User authentication is one of the hardest areas to grasp when first working with web APIs.
Without the benefit of a monolithic structure, we as developers have to deeply understand and
configure our HTTP request/response cycles appropriately. 
  
  
  
Django REST Framework comes with a lot of built-in support for this process, including
built-in TokenAuthentication. However developers must configure additional areas like user
registration and dedicated urls/views themselves. As a result, a popular, powerful, and secure
approach is to rely on the third-party packages dj-rest-auth and django-allauth to minimize
the amount of code we have to write from scratch.
  
  
  
  
  
  
  
</div>
