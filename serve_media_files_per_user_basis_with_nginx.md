# How to serve media files with nginx but users can reach only their files?

## You can do this with [X-Accel](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/)

### From [offical docs](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/):
> X-accel allows for internal redirection to a location determined by a header returned from a backend. <br>
> This allows you to handle authentication, logging or whatever else you please in your backend and then have NGINX handle serving the contents from redirected location to the end user, thus freeing up the backend to handle other requests. This feature is commonly known as X-Sendfile.

### Step 1: All requests for media should go throught a specific view.
```python
#myproject/urls.py
from myproject.views import media_access

urlpatterns = [
    ...,
    url(r'^media/(?P<path>.*)', media_access, name='media'),
]
```

### Step 2: Add the view to check access
```python
# myproject/views.py:
from django.http import HttpResponse
from django.http import HttpResponseForbidden

def media_access(request, path):
    """
    When trying to access :
    myproject.com/media/uploads/image.png

    If access is authorized, the request will be redirected to
    myproject.com/protected/media/uploads/image.png

    This special URL will be handle by nginx we the help of X-Accel
    """

    access_granted = False

    user = request.user
    if user.is_authenticated():
        if user.is_staff: # everything is granted for admin
            access_granted = True
        else:
            # Simple user only can acces their images
            user_files = [
                user.file,
                # add here more allowed files
            ]

            for file in user_files:
                if path == file.name:
                    access_granted = True

    if access_granted:
        response = HttpResponse()
        # Content-type will be detected by nginx
        del response['Content-Type']
        response['X-Accel-Redirect'] = '/protected/' + path
        return response
    else:
        return HttpResponseForbidden('Not authorized to access this media.')
```


## Step 3: Configure nginx

```nginx
# myproject/nginx.conf
upstream django_server {
  server localhost:8000;
}

server {
    listen 80;

    server_name myproject.com;

    server_name_in_redirect on;
    error_log /var/log/nginx/myproject-error.log crit;
    access_log  /var/log/nginx/myproject-access.log custom_combined;

    root /path/to/my_project/static;

    location ^~ /static/ {
        alias /path/to/my_project/static/;
    }

    location /protected/ {
        internal;
        alias /path/to/my_project/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://django_server;
        proxy_buffering off;
    }
}
```
