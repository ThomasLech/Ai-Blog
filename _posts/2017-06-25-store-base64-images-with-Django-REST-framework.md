---
<!-- layout: post -->
layout: post
title: Store base64 images using DRF
date: '2017-06-25 14:00 +0200'
published: true
comments: true
author: thomaslech
---

Hey, in this post we'll build a **web application** that can handle images taken using **[our mobile app][beautiful-camera-preview-with-cordova]** that we've previously built and store them in the database. For sake of simplicity I decided to do this using Django & Django-REST-framework.

After we're done with this, we'll perform **image processing** (in **[the next post][image-processing-for-text-recognition]**) and **build a predicive model**.

By now, we only have **base64** images that cannot be saved in the database yet. But, what is base64 and why we cannot save images represented in this format in the database?

### What is base64 format?
**[Base64][base64] is a way to represent binary data (stream of 1s and 0s) in an ASCII string format**.  
It was introduced in order to **avoid data corruption when transmitted in binary format** across a network.

**Examples of binary data corruption**:
- Some protocols may interpret your binary data as _control characters_ (like a modem).
- The underlying protocol might think that you've entered a special character combination  (like how FTP translates _line endings_).

So, after base64 image was **successfully transmitted** we can decode it back to binary data.  
There are two reasons why we are doing that:
- Later on, we want to perform **image processing** using [scikit-image][scikit-image-docs] and [opencv][opencv-docs] both of which naturally **only support binary input**.
- Django **ImageField requires image in binary format**. ImageField is an object that lets you save image files in the database.

Now, let's talk a little bit about the main topic of this post.

### What is Django?
[Django][Django-docs] is a free and open source **web application framework**, written in Python.
The role of Django boils down to **communicating with your app's database** and **serving particular data** to client along with static files.
Also, it has a set of handy components that let you for example handle **media uploads**, **form validation** or **user authentication**.

<!-- **(2)** [Django][Django-docs] is a free and open source **web application framework**, written in Python.
Django helps you develop websites faster and easier with its set of components:  
- A way to handle media uploads.
- User authentication.
- Form validation.


**(3)** [Django][Django-docs] is a free and open source **web application framework**, written in Python. A web framework is a set of components that helps you to develop websites faster and easier.  
**Most commonly used components** are:  
- A way to handle user authentication (signing up, signing in, signing out).
- Your website's managment panel.
- Form validation.
- A way to upload files. -->

### What is Django REST Framework?
[Django REST Framework][Django-REST-framework-docs](**DRF**) is a third-party app for Django.
DRF gives you a lot of convenience in **inspecting** and **managing** you database(via **"Browsable" API**).


## Client-side
Just to recall, we are sending cropped base64 image over the **http protocol** just like this:

```javascript
// Specify appropriate server endpoint address
// Ending slash in the address is necessary
$.post("http://server_address/",
    {
        // Data passed along a request
        image: cropped_img_base64
    },
    function(data, status, xhr) {
        // Success callback
        alert('Status: ' + status + '\Data: ' + data);
    }
)
.fail(function(error, status, xhr) {
    // Failure callback
    alert('Status: ' + status + '\nReason: ' + xhr);
});
```
We use [$.post][post-jquery] method which performs **asynchronous HTTP requests** to our server.  

## Server-side
Before we dive into building a web application, let's go through project setup first.

### DRF Setup
* **Install python 3.5**:  
[Windows x86-64 web-based installer][windows-x86-64-python-installer] or [Select & download other installer][other-python-installer]  

  > While installing Python make sure that option 'Add Python 3.5 to PATH' is checked.

* Create a new Django project named **digits_operators_recognizer**, then start a new app called **resolver**:

  ```python
  # Install virtualenv package
  # pip is a package management system used to install and manage software packages written in Python
  pip install virtualenv

  # Create the project directory
  mkdir digits_operators_recognizer
  cd digits_operators_recognizer

  # Create a virtualenv to isolate our package dependencies locally
  virtualenv env
  source env/bin/activate     # On Windows use `env\Scripts\activate`

  # Install Django and Django REST framework into the virtualenv
  pip install django djangorestframework

  # Set up a new project with a single application
  django-admin.py startproject digits_operators_recognizer .    # Note the trailing '.' character
  cd digits_operators_recognizer
  django-admin.py startapp resolver
  cd ..
  ```
* Add 'rest_framework' to your INSTALLED_APPS setting:

  ```python
  INSTALLED_APPS = (
      ...
      'rest_framework',
  )
  ```

### Code
Basically, building any Django project requires the same set of steps you need to take. This set of steps includes:
- **Define Django models and fields**.

  Generally, each model represents a single database table, e.g. users table.  
  Fields are table columns, e.g. username, password.

  ```python
  # digits_operators_recognizer/resolver/models.py

  # ...
  class Image(models.Model):

      # Define fields like below
      image = models.ImageField(upload_to='images/%Y/%m/%d/')
      # auto_now_add set to true makes this field filled with the date of entry creation
      timestamp = models.DateTimeField(auto_now_add=True)
      # we will add more fields here later

      # Django is using this method to display an object in the Django admin site
      def __str__(self):
      	 return self.image
  ```

- **Specify URL address and point it to a particular view**.

  Before users can visit a website, you must specify available URL addresses.

  ```python
  # digits_operators_recognizer/urls.py

  # ...
  from django.conf.urls import include
  from django.conf import settings
  from django.conf.urls.static import static

  urlpatterns = [
      # Here we include resolver app urls
      url(r'^api/', include('digits_operators_recognizer.resolver.urls')),
      # Address to admin site
      url(r'^admin/', admin.site.urls),
  ]

  # It lets us preview uploaded images through Browsable API
  if settings.DEBUG:
      urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
  ```

  Django project consists of **apps**. Each of these apps are supposed to be kept completely separate from each other. That's why they have their own set of **models**, **urls** and **views**.

  ```python
  # digits_operators_recognizer/resolver/urls.py

  # ...
  from digits_operators_recognizer.resolver import views
  from rest_framework.urlpatterns import format_suffix_patterns

  # Here we specify available URL addresses
  urlpatterns = [
      url(r'^images/$', views.ImageList.as_view()),
      url(r'^images/(?P<pk>[0-9]+)/$', views.ImageDetail.as_view()),
      url(r'^images/create/$', views.ImageCreate.as_view()),
  ]

  # Adding this lets you use filename extensions on URLs to provide an endpoint for a given media type.
  # For example you can get endpoint data in json representation or html static file
  urlpatterns = format_suffix_patterns(urlpatterns, allowed=['json', 'html'])
  ```

- **Build a view**

  View defines what will happen after someone enters corresponding URL address.

  ```python
  # digits_operators_recognizer/resolver/views.py

  import os
  from django.conf import settings

  # Our model
  from digits_operators_recognizer.resolver.models import Image

  # Our serializer
  from digits_operators_recognizer.resolver.serializers import ImageSerializer

  # DRF modules
  from rest_framework import status
  from rest_framework.request import Request
  from rest_framework.response import Response

  from rest_framework.generics import ListAPIView, RetrieveAPIView, CreateAPIView
  from rest_framework.permissions import IsAdminUser

  # OCR scripts
  from ocr import image, recognizer

  class ImageList(ListAPIView):

      'List all images'

      serializer_class = ImageSerializer
      permission_classes = (IsAdminUser,)
      queryset = Image.objects.all()

  class ImageDetail(RetrieveAPIView):

	  'Retrieve an image instance'

	  serializer_class = ImageSerializer
	  permission_classes = (IsAdminUser,)
	  queryset =  Image.objects.all()

  class ImageCreate(CreateAPIView):

      'Create a new image instance'

      serializer_class = ImageSerializer

      def post(self, request):

          serializer = ImageSerializer(data=request.data)
          if serializer.is_valid():

              # Save request image in the database
              serializer.save()

              return Response(serializer.data, status=status.HTTP_201_CREATED)

          return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
  ```

Last but not least, I would like to introduce you to **DRF serializers**.

**Serializers** allow complex data such as **querysets** and **model instances** to be converted to native Python datatypes that can then be easily rendered into **JSON**, **XML** or **other content types**.  
Serializers also provide deserialization, allowing parsed data to be converted back into complex types, after first validating the incoming data.

In other words, a serializer translates given queryset to **human readable format** and vice versa.

For sake of clarity, I decided to create a separate file named **_serializers.py_** inside **resolver directory** where we will declare our serializer.

```python
# digits_operators_recognizer/resolver/serializers.py

class ImageSerializer(serializers.HyperlinkedModelSerializer):
    image = Base64ImageField(
        max_length=None, use_url=True,
    )

    class Meta:
        model = Image
        fields = ('pk', 'image', 'timestamp')
```

### Running server
```bash
# Sync the database - propagate changes you made to your models into your database schema
python manage.py migrate

python manage.py runserver
```


## Full code
Visit my repo: [digits_operators_recognizer][digits-operators-recognizer-repo]

## Conclusion
In the **[next post][image-processing-for-text-recognition]** we will perform **image processing** in order to extract **relevant features** that can be fed into **predicive model**.


[beautiful-camera-preview-with-cordova]: http://blog.mathocr.com/2017/06/09/beautiful-camera-preview-with-cordova.html
[image-processing-for-text-recognition]: http://blog.mathocr.com/2017/06/25/image-processing-for-text-recognition.html

[cordova-camera-preview-example]: https://github.com/ThomasLech/cordova-camera-preview-example
[web-api]: https://en.wikipedia.org/wiki/Web_API#Server_side
[base64]: https://en.wikipedia.org/wiki/Base64
[scikit-image-docs]: http://scikit-image.org/
[opencv-docs]: http://opencv.org/
[Django-REST-framework-docs]: http://www.django-rest-framework.org/
[Django-docs]: https://www.djangoproject.com/
[ajax-jquery]: http://api.jquery.com/jquery.ajax/
[post-jquery]: https://api.jquery.com/jquery.post/
[jquery]: https://jquery.com/
[digits-operators-recognizer-repo]: https://github.com/ThomasLech/digits_operators_recognizer
[windows-x86-64-python-installer]: https://www.python.org/ftp/python/3.5.0/python-3.5.0-amd64-webinstall.exe
[other-python-installer]: https://www.python.org/downloads/release/python-350/
