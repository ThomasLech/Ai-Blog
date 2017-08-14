---
<!-- layout: post -->
layout: post
title: Beautiful camera preview with cordova
date: '2017-06-09 22:21:03 +0200'
published: true
comments: true
author: thomaslech
---


Hi, recently I wanted to build a mobile app for my ML project. As usual I was planning to acomplish this with Java. I started searching through Android forums and its community to find out more about manipulating camera preview. However, to my surprise I found it **hard to fully understand tens or hundreds lines of tedious code**. I felt frustrated at first. So searching desperately for easier solutions I came across **Cordova framework for developing hybrid mobile apps** and its cool plugin for customizing camera preview.


In fact there are several frameworks for building hybrid mobile apps.


## Most popular frameworks
- [IONIC][ionic-docs]  
**Ionic has strong AngularJS influence**. So if you're not familiar with AngularJS, you might want to try other frameworks.
- [CORDOVA][cordova-docs]  
Cordova is an open source project supported by Apache. This is the **simplest and fastest** framework to use.
- [PHONEGAP][phonegap-docs]  
PhoneGap is Adobe's distribution of Cordova. PhoneGap is powered by Cordova but has a separate command line tool.  
[PhoneGap Build][phonegap-build-docs] is a service provided by Adobe. With PhoneGap Build, you upload your HTML, CSS and JavaScript to Adobe's servers and they build native applications for you.  
The major benefit is that you **don't have to have the native SDKs installed on your computer**.  
**This lets you do things like build iOS application from Windows**.

> **NOTE**: You definitely should read about [cons of building a hybrid app][cons-hybrid] before building any other native apps.



## Easy Cordova setup
```bash
# Install nodejs
# Nodejs is a package manager, distributing javascript packages
sudo apt-get install nodejs

# Install cordova using nodejs
npm install -g cordova

# Create a new cordova project
cordova create path/to/myApp

# Add platforms you want to build your app for
# List available platforms: cordova platform
cordova platform add <platform_name>

# Add camera preview plugin
cordova plugin add cordova-plugin-camera-preview
```

> **NOTE**: For some reason emulator appear not to take cordova-camera-preview plugin into consideration.  
> However, **building the app directly on mobile device does the trick**.



## Camera preview

**_index.html_**
```html
...
<body>
    <script type="text/javascript" src="cordova.js"></script>

    <!-- jQuery for http POST request -->
    <script src="https://code.jquery.com/jquery-1.12.4.js"></script>

    <!-- crop.js contains our crop function -->
    <script type="text/javascript" src="js/crop.js"></script>
    <script type="text/javascript" src="js/index.js"></script>
</body>
```
We remove all redundant code from body section **except script tags**.  
Notice **_crop.js_** file. Later in this post we will define crop function in there.


**_index.js_**
```javascript
...
onDeviceReady: function() {
    // Method below REQUIRES elements we removed from body in index.html
    // So we comment it out.
    // this.receivedEvent('deviceready');

    let options = {
        x: 0,
        y: 0,
        width: window.screen.width,
        height: window.screen.height,
        camera: CameraPreview.CAMERA_DIRECTION.BACK,  // Front/back camera
        toBack: true,   // Set to true if you want your html in front of your preview
        tapPhoto: false,  // Tap to take photo
        tapFocus: true,   // Tap to focus
        previewDrag: false
    };

    var flash_mode = 'off';
    // Take a look at docs: https://github.com/cordova-plugin-camera-preview/cordova-plugin-camera-preview#methods
    CameraPreview.startCamera(options);


    // Create a rectangle & buttons
    var rect = document.createElement('div');
    var take_pic_btn = document.createElement('img');
    var flash_on_btn = document.createElement('img');
    var flash_off_btn = document.createElement('img');

    // You must specify path relative to www folder
    take_pic_btn.src = 'img/btn_icon_mini.png';
    flash_on_btn.src = 'img/flash_on.svg';
    flash_off_btn.src = 'img/flash_off.svg';

    // Add styles
    rect.className += 'rect_class';
    take_pic_btn.className += 'btn_class';
    flash_on_btn.className += 'btn_class';
    flash_off_btn.className += 'btn_class';

    take_pic_btn.className += ' take_pic_class'
    flash_on_btn.className += ' flash_class'
    flash_off_btn.className += ' flash_class'

    // Hide flash_off btn by default
    flash_off_btn.style.visibility = 'hidden';

    // Append to body section
    document.body.appendChild(rect);
    document.body.appendChild(take_pic_btn);
    document.body.appendChild(flash_on_btn);
    document.body.appendChild(flash_off_btn);

    // Get rectangle coordinates
    var rect_coords = rect.getBoundingClientRect();
    var x_coord = rect_coords.left, y_coord = rect_coords.top;

    take_pic_btn.onclick = function(){
        // Get rectangle size
        var rect_width = rect.offsetWidth, rect_height = rect.offsetHeight;

        CameraPreview.takePicture(function(base64PictureData) {

            // We pass width, height, x and y coordinates of our rectangle to crop method
            // At the very end, crop methods send cropped image to server
            var cropped_img = crop(base64PictureData, rect_width, rect_height, x_coord, y_coord, function(cropped_img_base64) {

                // Ending slash is necessary
                $.post("server_address/",
                    {
                        // Data sent along with a request
                        image: cropped_img_base64
                    },
                    function(data, status, xhr) {
                        // Success callback
                        alert('Status: ' + status + '\nData: ' + data);
                    }
                )
                .fail(function(error, status, xhr) {
                    // Failure callback
                    alert('Status: ' + status + '\nReason: ' + xhr);
                });

            });
        });
    };

    flash_on_btn.onclick = function() {
        flash_mode = 'on';
        flash_off_btn.style.visibility = 'visible';
        flash_on_btn.style.visibility = 'hidden';

        CameraPreview.setFlashMode(flash_mode);
    }

    flash_off_btn.onclick = function() {
        flash_mode = 'off';
        flash_off_btn.style.visibility = 'hidden';
        flash_on_btn.style.visibility = 'visible';

        CameraPreview.setFlashMode(flash_mode);
    }
},
...
```
First, locate **onDeviceReady** method defined inside **_index.js_**.  
Within **onDeviceReady** body, we start camera activity with specific options set.  
Next, we create a rectangle & submit button. When button is clicked we **crop area set by the rectangle**.



**_crop.js_**
```javascript
var crop = function(base64PictureData, rect_width, rect_height, x_coord, y_coord, callback) {

    // image variable will contain ORIGINAL image
    var image = new Image();

    // canvas variable will contain CROPPED image
    var canvas = document.createElement('canvas');
    var ctx = canvas.getContext('2d');

    // Load original image onto image object
    image.src = 'data:image/png;base64,' + base64PictureData;
    image.onload = function(){

        // Map rectangle onto image taken
        var x_axis_scale = image.width / window.screen.width
        var y_axis_scale = image.height / window.screen.height
        // INTERPOLATE
        var x_coord_int = x_coord * x_axis_scale;
        var y_coord_int = y_coord * y_axis_scale;
        var rect_width_int = rect_width * x_axis_scale;
        var rect_height_int = rect_height * y_axis_scale

        // Set canvas size equivalent to cropped image size
        canvas.width = rect_width_int;
        canvas.height = rect_height_int;

        ctx.drawImage(image,
            x_coord_int, y_coord_int,           // Start CROPPING from x_coord(interpolated) and y_coord(interpolated)
            rect_width_int, rect_height_int,    // Crop interpolated rectangle
            0, 0,                               // Place the result at 0, 0 in the canvas,
            rect_width_int, rect_height_int);   // Crop interpolated rectangle

        // Get base64 representation of cropped image
        var cropped_img_base64 = canvas.toDataURL();

        // Now we are ready to send cropped image TO SERVER
        callback(cropped_img_base64);

        return cropped_img_base64;
    };
};
```

Since the size of image taken is **completly different** from camera preview size, we need to map rectangle onto image taken.


**_index.css_**
```css
/* index.css */

div.rect_class {
    width: 280px;
    height: 100px;

    /* Make inner part of rectangle transparent */
    background-color: rgba(255, 255, 255, 0);

    /* Center vertically AND horizontally */
    /* position attribute can be fixed too*/
    position: absolute;
    left: 0; right: 0;
    top: 0; bottom: 0;
    margin: auto;

    /* This to solve "the content will not be cut when the window is smaller than the content": */
    max-width: 100%;
    max-height: 100%;
    overflow: auto;

    /* COOL BORDER */
    border-width: 20px;
    border-style: solid;
    /* You need to place border.png into img folder */
    border-image: url(../img/border.png) 50 round;

    /* SHADOW EFFECT darkens everything outside rectangle */
    box-shadow: 0 0 500px 5000px rgba(0, 0, 0, 0.4);
}

img.btn_class {
    width: 75px;
    height: 75px;

    /* Center horizontally */
    /* position attribute can be fixed too*/
    position: absolute;
    margin: auto;
}

img.flash_class {
    top: 20px; right: 20px;
}

img.take_pic_class {
    bottom: 20px;
    left: 0px; right: 0px;
}
```

#### **Status bar**
Our app would be looking even better with **customized status bar**. Cordova enables us customizing status bar with **cordova-plugin-statusbar**. For example we can change its background color.

**Example of statusbar customization**:  
- Install plugin: `cordova plugin add cordova-plugin-statusbar`.
- Head over to **_config.xml_** inside your project's root directory.
- Insert `<preference name="StatusBarBackgroundColor" value="HEX_RGB_CODE" />`.

For more info visit [statusbar plugin docs][statusbar-plugin-docs].

#### **Resize rectangle**
You might also make elements **draggable/resizable** using **jQuery UI Touch Punch**.  
Visit [their docs][jquery-ui-touch-punch] for installation guide and examples.



## Results
![screenshot](https://user-images.githubusercontent.com/22115481/27963916-c4c395aa-6336-11e7-90a4-ea54eb3d14c4.jpg)


## Full code
Visit my repo: [cordova-camera-preview-example][cordova-camera-preview-example]


## Conclusion
Now, that we have **cropped base64 image**, we want to do something cool with it. Predicion of **digits combined with arithmetic operators** found in an image sounds cool.


We want our server to serve predictions, so that clients from around the world can use our recognizer.


In the **[next post][store-base64-image-with-Django-REST-Framework]**, I'll show you how to build **web application** that can handle requested images and store them on a server using **Django-REST-framework**. But by now, I wish you a great day :)


[ionic-docs]: https://ionicframework.com/
[cordova-docs]: http://cordova.apache.org/
[phonegap-docs]: https://phonegap.com/
[phonegap-build-docs]: https://build.phonegap.com/
[cons-hybrid]: http://blog.icreon.us/launch/native-vs-hybrid-development
[camera-preview-plugin-docs]: https://github.com/cordova-plugin-camera-preview/cordova-plugin-camera-preview
[statusbar-plugin-docs]: https://cordova.apache.org/docs/en/latest/reference/cordova-plugin-statusbar/#statusbar
[jquery-ui-touch-punch]: http://touchpunch.furf.com/
[cordova-camera-preview-example]: https://github.com/ThomasLech/cordova-camera-preview-example
[scikit-image-docs]: http://scikit-image.org/
[opencv-docs]: http://opencv.org/
[store-base64-image-with-Django-REST-Framework]: http://blog.mathocr.com/2017/06/25/store-base64-images-with-Django-REST-framework.html
