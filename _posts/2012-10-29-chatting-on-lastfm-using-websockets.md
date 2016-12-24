---
layout: post
title: Chatting on Last.fm using websockets
image: '/images/qop-lastfm.png'

---

Adding a Chat box on Last.fm is easier than you think. Here's a quick hack.

![qop chat](/images/qop-lastfm.png "qop chat screenshot")


The hack
--------

As shown before, it's relatively easy to embed a [chat box on twitter](/2012/05/19/chatting-on-twitter-with-pusher.html).
Doing the same for [last.fm](http://last.fm) is trivial. First, we need to add _omniauth\_lastfm_ in the _Gemfile_ of our Sinatra app.

{% highlight ruby %}
    gem 'omniauth-lastfm'
{% endhighlight %}

We need to add also the _require_ in the app file:

{% highlight ruby %}
    require 'omniauth-lastfm'
{% endhighlight %}

Now we have to define the OAuth callback for Lastfm:

{% highlight ruby %}
  get '/auth/lastfm/callback' do
    auth = request.env["omniauth.auth"]

    user = User.first_or_create({ :uid => auth["uid"]}, {
      :uid => auth["uid"],
      :name => auth["info"]["name"],
      :nickname => auth["info"]["name"],
      :created_at => Time.now })

    session[:user_id] = user.id # User logged in
    redirect 'http://last.fm'
  end
{% endhighlight %}

Chrome Extension
----------------

We add a **Chrome Extension** that automatically loads up the javascript and opens the chat box. The _manifest.json_ is:

{% highlight JSON %}
    {
       "content_scripts": [ {
          "js": [ "qop.js" ],
          "matches": [ "http://*.last.fm/*", "https://*.last.fm/*" ],
          "run_at": "document_end"
       } ],
       "description": "Last.fm Chat",
       "name": "qop.im",
       "version": "1.0",
       "manifest_version": 2
    }
{% endhighlight %}

And the **qop.js** is:

{% highlight javascript %}
    if (window.top === window) {
        (function(){
            link = document.createElement('link');
            link.href = 'http://YOURSERVER/application.css';
            link.type = 'text/css';
            link.rel = 'stylesheet';
            document.getElementsByTagName('head')[0].appendChild(link);
            var script = document.createElement('script');
            script.type = 'text/javascript';
            script.async = true;
            script.src = 'http://YOURSERVER/application.js';
            document.getElementsByTagName('head')[0].appendChild(script);})();
    }
{% endhighlight %}

That's all. Give it a spin if you'd like, but remember that it's _alpha_ code.
Here's the Chrome extension: [Chat on Last.fm](https://chrome.google.com/webstore/detail/qopim/afbllpelodekoheogfidhgncbfpendlj)