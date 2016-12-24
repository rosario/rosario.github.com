---
layout: post
title: HTML5 Video Streaming over WebSockets using Pusher
image: '/images/pusher-logo.png'
---


Using the new **getUserMedia** it's possible to send a stream of images over **websockets**. An experiment with Node.js and Pusher Pipe.



Overview
--------

There's a new feature in the latest Chrome browser's (Chrome 18 and 19, and Canary 21) called **getUserMedia**.
It's still an experimental feature and it's hidden behind _chrome://flags_

> **Enable Media Source API on `<video>` elements.**
  Enable experimental Media Source API on the video elements. This API allows JavaScript to send media data directly to a video element.


It's a trivial task to grab a screenshot using **video** and **canvas**, and it's also very easy to send
this image over websockets using [pusher](http://www.pusher.com "pusher").



Express.js and Pusher Pipe
-------------------

We'll use [Express.js](http://expressjs.com/ "Express.js") and **Pusher Pipe**. A quick
note on [pusher pipe](http://pusher.com/docs/pipe "pusher pipe"), instead of sending requests using a typical
_REST_ API, using the _Pipe_ we can enable our Node server and the browsers to communicate with Pusher services
using websockets.

{% highlight html %}
    <div class="center">
      <img src="/images/pusher-logo.png" alt="pusher logo">
    </div>
{% endhighlight %}

The following code is written in _Coffescript_. We tell our server to render two pages, __stream__ is used
to send a stream of images, and __mirror__ is used to receive the images and display them:

{% highlight coffeescript %}
    # Express.js. Serving /stream and /mirror
    express = require('express')

    app = express.createServer(express.logger())
    app.set('view engine', 'jade');
    app.use('/js', express.static(__dirname + '/js'));

    port = 9000 || process.env.PORT || 3000;
    app.listen port, ->
      console.log("Listening on " + port);

    app.get '/stream', (req, res) ->
      res.render('stream');

    app.get '/mirror', (req, res) ->
      res.render('mirror');
{% endhighlight %}


Now we connect to the _Pipe_. The following code sends to the channel _mirror_ the data received on the channel _stream_

{% highlight coffeescript %}
    # Pusher Pipe. Streaming data over WebSockets
    Pipe = require('pusher-pipe')
    client = Pipe.createClient {
      key: 'your key'
      secret: 'your secret'
      app_id: 'your app id'
      debug: true
    }

    client.subscribe ['socket_message']
    client.channel('stream').on 'event:frame', (socketId, data) ->
      client.channel('mirror').trigger 'frame', {message: 'Sending frame', dataUrl: data.dataUrl}

    client.connect()
{% endhighlight %}

Note that the urls _/stream_ and _/mirror_ are served with Express, and they respond to the usual http GET
method. The channels 'stream' and 'mirror' instead are opened with Pusher.


Magic Mirror on the Wall...
---------------------------


Let's have a look first at the page receiving the stream of images. The user hits _/mirror_, the server
renders this **jade** template:


{% highlight jade %}
      h1 Mirror

      img(id="screenshot", src="")
      canvas(id="screenshot-canvas", style="display:none;")
      script(src="js/player.js")
{% endhighlight %}

We use the class __Player__ to connect with the websocket and update the \<img\> with a stream of data:

{% highlight coffeescript %}
      document.addEventListener('DOMContentLoaded', ->
        new Player();
      , false);

      class Player
        constructor: ->
          @img = document.querySelector('#screenshot')
          Pusher.host = "ws.darling.pusher.com"
          Pusher.log = (message) ->
            if (window.console && window.console.log)
              window.console.log(message)

          @pusher = new Pusher('YOURKEY')

          @mirror = @pusher.subscribe 'mirror'
          @mirror.bind 'frame', (data) =>
            @img.src = data.dataUrl;
{% endhighlight %}

The idea is to update the **src** of an img tag with the data sent over websockets, and it's simply done
by binding the _@mirror_ channel to the _frame_ event.

Meanwhile our Hero...
---------------------

The _/stream_ page contains the following template:

{% highlight jade %}
        h1 Video

        video(autoplay)
        img(id="screenshot", src="")
        canvas(style="display:none;")

        button(id="start") Capture
        button(id="stop") Stop

        script(src="js/stream.js")
{% endhighlight %}

We use two buttons, one to start the streaming, and one to stop it. The coffescript code is the following:

{% highlight coffeescript %}
      document.addEventListener('DOMContentLoaded', ->
        new VideoStream();
      , false);

      class VideoStream
        constructor: ->
          @video = document.querySelector('video')
          @video.addEventListener('click', @snapshot, false)

          @button = document.querySelector('#start')
          @button.addEventListener('click', @snapshot, false)

          @stop = document.querySelector('#stop')
          @stop.addEventListener('click', @stopRefresh, false)

          @canvas = document.querySelector('canvas')
          @ctx = @canvas.getContext('2d')
          @getUserMedia()

          Pusher.host = "ws.darling.pusher.com"
          Pusher.log = (message) ->
            if (window.console && window.console.log)
              window.console.log(message)

          @pusher = new Pusher('YOURKEY')
          @channel = @pusher.subscribe 'stream'

        sizeCanvas: ->
          setTimeout( =>
            @canvas.width = @video.videoWidth;
            @canvas.height = @video.videoHeight;
          , 50)

        refresh: =>
          @ctx.drawImage(@video, 0, 0)
          dataUrl = @canvas.toDataURL('image/webp')
          @channel.trigger 'frame',  dataUrl: dataUrl

        stopRefresh: =>
          console.log('stop timer')
          clearTimeout(@timer)

        snapshot: =>
          @timer = setInterval( @refresh, 250)

        getUserMedia: ->
          version = parseInt(window.navigator.appVersion.match(/Chrome\/(.*?) /)[1].split('.')[0]);
          if navigator.webkitGetUserMedia
            if (version < 21)
                navigator.webkitGetUserMedia('video', (stream) =>
                  @video.src = window.webkitURL.createObjectURL(stream);
                  @sizeCanvas()
                  @button.textContent = 'Take Shot'
                , @onFailSoHard)
            else
                navigator.webkitGetUserMedia({video:true}, (stream) =>
                  @video.src = window.webkitURL.createObjectURL(stream)
                  @sizeCanvas()
                  @button.textContent = 'Take Shot'
                , @onFailSoHard)
          else
           console.log 'Error with getUserMedia()'
{% endhighlight %}

The **getUserMedia** is a wrapper around webkitGetUserMedia. We need it because Chrome 18/19  and Chrome 21
have a slightly different syntax.

After the user clicks the button, we start a timer and we refresh the image taken from the video and send
this data to the _@channel_.


Fin
---

The first problem lies in the way images are encoded, they are Base64 images. Ideally we could send
binary data over a websocket. Pusher also has a rate limit on the number of messages per second.

Can we use websockets for video streaming? Not with this setup, after all it was just an experiment.


Source Code and Credits
-----------------------


The source code can be found here [videostream-pusher-pipe](https://github.com/rosario/videostream-pusher-pipe "videostream pusher pipe")

This experiment borrowed code from [smartjava.org](http://www.smartjava.org/content/face-detection-using-html5-javascript-webrtc-websockets-jetty-and-javacvopencv "Face detection using HTML5, javascript, webrtc, websockets, Jetty and OpenCV")
and from [html5rocks.com](http://www.html5rocks.com/en/tutorials/getusermedia/intro/ "CAPTURING AUDIO & VIDEO IN HTML5")




