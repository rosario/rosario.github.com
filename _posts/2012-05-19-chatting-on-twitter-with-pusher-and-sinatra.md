---
layout: post
title: Chatting on Twitter
excerpt: |-
  #### Can we embed a chat widget on **Twitter**? Yes, with few tricks and **Pusher** we can create a widget and chat with our friends.
  ![qop chat](/images/qop2.png "QoP Chat")
  
---


### Can we embed a chat widget on **Twitter**? Yes, with few tricks and **Pusher** we can create a widget and chat with our friends.



Overview: qop qop qop
--------

This is a short description of **qop**, a simple and unpretentious chat widget that can be embedded on Twitter.
It is written in Ruby and Coffescript. Pusher is used to send chat messages and notifications. Webservices are 
provided with Sinatra. Authentication is done via Omniauth. Data models are written in Datamapper, and Handlebars 
is used for templates. Finally a custom Sprockets process will glue everything together. As easy as that!


![qop chat](/images/qop2.png "QoP Chat")



The bookmarklet
---------------

A bookmarklet is used to load the widget from our server, and this is the code:

    {% highlight javascript %}
    
    	(function(){
    		link = document.createElement('link');
    		link.href = 'https://YOURQOPSERVER.herokuapp.com/assets/application.css';
    		link.type = 'text/css';
    		link.rel = 'stylesheet';
    		document.getElementsByTagName('head')[0].appendChild(link);

    		var script = document.createElement('script');
    		script.type = 'text/javascript';
    		script.async = true;
    		script.src = 'https://YOURQOPSERVER.herokuapp.com/assets/application.js';
    		document.getElementsByTagName('head')[0].appendChild(script);
    	})();
    {% endhighlight %}


Note that it will be easy to create a Chrome Extension, just add a _manifest.json_ and Chrome will do the rest. The 
file _application.js_ contains the entire widget code.



Loading, please wait
-----------------

We'll load jQuery and wait until is completely loaded. We also need to load Pusher library, and finally load the **qop**.

    {% highlight coffeescript %}
    
    	done = false
    	script = document.createElement('script')
    	script.src = 'https://ajax.googleapis.com/ajax/libs/jquery/1.7.1/jquery.min.js'

    	script.onload = script.onreadystatechange = ->
    	  if (!done and (!@readyState or @readyState =='loaded' or @readyState =='complete'))
    	    done = true

    	    window.qop$ = jQuery.noConflict();
    	    window.qop$.support.cors = true

    	    qop$.getScript("https://d3dy5gmtp8yhk7.cloudfront.net/1.11/pusher.min.js")
    	      .done (script) ->
    	        console.log 'loading app...'
    	        app = new QoP.App()
    	      .fail (jqxhr, settings, exception) ->
    	        console.log 'failed'
	
    	document.getElementsByTagName('head')[0].appendChild(script)
    {% endhighlight %}



The **App** class doesn't do much, just initialise Pusher with the right credentials, and create the contact list.


{% highlight coffeescript%}

	class QoP.App
	  constructor: () ->
	    qop$("<div id='qop'></div>").appendTo 'body'
	    Pusher.channel_auth_endpoint = 'https://YOURQOPSERVER.herokuapp.com/pusher/auth'
	    Pusher.channel_auth_transport = 'jsonp'
	    QoP.pusher = new Pusher 'APPKEY'
	    contacts = new QoP.Contacts('contact_list', 'Contact List')
	
{% endhighlight %}



You're a bit too Pushy
----------------------


**Pusher** provides a great **websockets** service and API, and we'll use websockets to send chat messages. The browsers
communicates with typical _AJAX_ requests with our server.


![qop chat](/images/pusher.png "QoP Chat")


CORS stands for Cross-Origin Resource Sharing, and it does what it says. Modern browsers support CORS,
and we only care about modern browsers. We could use JSONP, but that is really old school (and CORS is more elegant).

The browser makes _CORS_ requests to our server using jQuery:

    {% highlight coffeescript %}
    
      class QoP.Server
        @sendRequest: (service_name, data = null, callback = null) ->
          qop$.support.cors = true
          response = qop$.ajax "https://YOURQOPSERVER.herokuapp.com/#{service_name}",
            type: "POST"
            data: data
            contentType: "application/json; charset=utf-8"
            dataType: 'json'
            beforeSend: ( xhr ) ->
                xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')
                xhr.withCredentials = true
            success: (data) ->
              callback(data) if callback
            error: (XMLHttpRequest, textStatus, errorThrown) ->
              console.log "Error => #{textStatus}"
              console.log errorThrown
            xhrFields:
               withCredentials: true
            crossDomain: true
          return
    {% endhighlight %}

We force jQuery to use **cors**, by setting a flag to true, we also need to pass *xhrFields* 
and set **withCredentials** to true. It shouldn't be needed, but just in case we also set **crossDomain** 
to true. It is a bit redundant, but there have been cases it won't work otherwise. If you know
any better way of doing this, please let me know.

The server needs to know we want to accepts request from _twitter.com_. Since the server is a **Sinatra** 
application we use the **rack-cors** gem. (You want to include localhost for testing). 



    {% highlight ruby %}
  
    # This goes in the Gemfile
    gem 'rack-cors', :require => 'rack/cors'
    
    # This goes in the sinatra app.rb file
    require 'rack/cors'
    
    use Rack::Cors do
      allow do
        origins 'http://localhost:5000', 'https://twitter.com'
        # /messages is used to send messages to a channel
        resource '/messages',  
            :methods => [:post, :options], :headers => :any, :credentials => true
            
        # /messages_all is used to send messages to a public channel            
        resource '/messages_all', 
            :methods => [:post, :options], :headers => :any, :credentials => true
            
        # /chat_session gives informations about the connect user            
        resource '/chat_session', 
            :methods => [:post, :options], :headers => :any, :credentials => true
            
        # /chat_request ask the server a private channel to connect two users
        resource '/chat_request', 
            :methods => [:post, :options], :headers => :any, :credentials => true
            
        # pusher/auth is used to authenticate pusher presence channels
        resource '/pusher/auth', 
            :methods => [:get, :options],  :headers => :any, :credentials => true
      end
    end
    {% endhighlight %}


Each url correspond to a service we need to provide to *qop*.

My Friends are your Friends
---------------------------

Let's see how to populate the contact list


    {% highlight coffeescript%}
    
    class QoP.Contacts extends QoP.Box
      constructor: (id, name) ->
        @presenceChannel = null
        @friends = null   # List of online friends
        @uid = null       # twitter uid
        @online = null    # if the user is online/offline
        @nickname = null  # the user nickname
        
        @bind 'qop:init_presence', @initPresenceChannel
        
        QoP.Server.sendRequest 'chat_session', null, (data) =>
          @friends = data.friends
          @uid = data.uid
          @online = data.online == "true"
          @nickname = data.nickname
          content = qop$(JST['content'](id: "content_#{id}"))
          @box.append content 
          @trigger 'qop:init_presence'
    {% endhighlight%}
    
The **chat\_session** method returns a JSON object with user's online _@friends_,  _@uid_ 
(unique identifier provided by Twitter) and  _@nickname_. Note, __online__ means the user has  been 
authenticated with our server. (The **JST** variable is generated by Sprockets and it holds our 
handlebars templates).


The **initPresenceChannel** function is called (via a trigger) if *chat\_session* request is succesful.


    {% highlight coffeescript%}
    
    initPresenceChannel: ->
      if @online
        @presenceChannel = QoP.pusher.subscribe("presence-#{@uid}")
        @presenceChannel.bind 'pusher:subscription_succeeded', @presenceSucceeded
        @presenceChannel.bind 'friend_status', @updateFriendStatus
        @presenceChannel.bind 'create_chat', @createChat
      else
        @box.find('.contactlist').append qop$(JST['signup']({redirect: redirectUrl}))
        @trigger 'box:raise'
      return
    {% endhighlight%}
    
If the user is online (authenticated) we subscribe the user to **his own** presence channel. 
If the user is not  online, we will show a signup link inside the widget, asking the user to authenticate.
Authentication is done via the **omniauth\-twitter** gem.


Presence Channels with Pusher
-----------------------------

Each user has got a _personal_ presence channel. We use this channel to communicate events to the user. 
For example, we use the presence channel to let a user when a friend *joins* or *leaves* 
the chat. Pusher makes it really easy to know about these events via **webhooks**. Here's the server code:

    {% highlight ruby %}
    
    post '/webhooks' do
      webhook = Pusher::WebHook.new(request)
      if webhook.valid?
        webhook.events.each do |event|
          uid = event["channel"].split('-').last
          user = User.first(:uid=>uid)
          if user
            case event["name"]
              when 'channel_occupied'
                user.online!
              when 'channel_vacated'
                user.offline!
            end
            user.online_friends.each do |friend|
              Pusher["presence-#{friend.uid}"].trigger_async('friend_status',{ 
                  :uid => user.uid, 
                  :nickname => user.nickname, 
                  :online =>user.online
              })
            end
          end
        end
      else
        status 401
      end
      return
    end
    {% endhighlight %}
  
When the user occupies the channel, the user status will be set to _online_, otherwise the user is _offline_. 
We use "cowboy" notifications to send these status changes, for each online friend we send a _friend\_status_ 
notification. Having said that, the code is highly inefficient, since we shouldn't do that, It would be better 
to send notifications outside the http request cycle... Anyway...


Look Ma, I have a friend
------------------------

At this point is user is authenticated and his presence channel is working. It's time to show/hide friends
and to open a chat box when we click on their nickname.
    
    {% highlight coffeescript%}
    
    presenceSucceeded: =>
      @trigger 'box:raise'
      for user in @friends
        console.log "Render user: #{user.nickname}"
        @addFriend(user)
      return
    
    updateFriendStatus: (user) =>
      if user.online then @addFriend(user) else @removeFriend(user)
      return    
    
    addFriend: (user) ->
      check = @box.find("#contact_#{user.uid}").length == 0
      if check
        friend = qop$(JST['friend'](user: user))
        @box.find('.friends_list').append friend
        friend.bind 'click', {user: user}, (event) =>
          QoP.Server.sendRequest('chat_request', {uid: event.data.user.uid})
      return

    removeFriend: (user) ->
      @box.find(".friends_list p#contact_#{user.uid}").remove()
      return
    {% endhighlight%}
    
Adding an online friend is easy, we check if the friend isn't already shown, then we fill a template 
with the relevant data, and we append this template to the contact list. To remove an offline 
friend is even easier.


Now we need to create a chat box. Pusher is here to help us again. We bind the click event so that 
when we click on the nickname the browser does an _AJAX_ **chat\_request** to our server.

    {% highlight ruby %}
    
    post '/chat_request' do
      someone = User.first(:uid=> params[:uid])
      if someone and someone.online and current_user.friend?(someone)
        cuid = current_user.uid
        suid = someone.uid
        
        # A chat between two user is created using their twitter uids
        if cuid < suid
          chat_channel = "chat-channel-#{cuid}-#{suid}"
        else
          chat_channel = "chat-channel-#{suid}-#{cuid}"
        end
        
        Pusher["presence-#{current_user.uid}"].trigger_async('create_chat', {
          :uid =>someone.uid,
          :nickname=>someone.nickname,
          :channel_name => chat_channel
        })
        
        Pusher["presence-#{someone.uid}"].trigger_async('create_chat',{
          :uid => current_user.uid,
          :nickname =>current_user.nickname,
          :channel_name => chat_channel
        })
        
      end
      content_type :json
      {:request => 'sent'}.to_json
    end
    {% endhighlight %}

If our friend is online we create a new channel. This channel holds the chat between the two users.
We use Pusher to trigger a **create\_chat** event in the browsers.



    {% highlight coffeescript%}  
    
    createChat: (data) =>
      channel_name = data.channel_name
      friend =  uid: data.uid, nickname: data.nickname
      user = uid: @uid, nickname: @nickname
      panel =  new QoP.Panel(channel_name, friend, user)
      return
    {% endhighlight%}
    
    
The _createChat_ creates a new Panel, and it allows us to chat with our friend.


Panel Discussions
-----------------

The Panel is where we type our messages and chat with our friend. Each Panel has a pusher channel associated 
with it, and this channel is used to send chat messages. 


    {% highlight coffeescript%}
    
    class QoP.Panel extends QoP.Box
      constructor: (channel_name,friend = {}, user = {}) ->
        @channel = null
        @channel_name = channel_name
        @name = null
        @user = user
        @friend = friend
    {% endhighlight%}
    

To create the Panel, we render the template and append a text area. 


    {% highlight coffeescript%}
    
    createBox: ->
      panel = qop$(JST['panel'](id: "panel_#{@friend.uid}"))
      @box.append panel
      textarea = qop$(JST['textarea'](uid: @friend.uid))
      @box.append textarea
    {% endhighlight %}
    
We also need to subscribe the users to the new channel. 

    {% highlight coffeescript%}
    
    createChannel: ->
      if not @channel?
        @channel = QoP.pusher.subscribe(@channel_name)
        @channel.bind 'pusher:subscription_succeeded', console.log("User subscribed")
        @channel.bind 'pusher:subscription_error', console.log('User already subscribed')
        @channel.bind 'new_message', @messageReceived
      @channel
    {% endhighlight%}



Twitter has keyboard shortcuts, we need to disable them when the user is typing. After the user hit _enter_
we send the message to our server:

    {% highlight coffeescript%}
    
    enableTextBox: ->
      textBox = @box.find('.chatboxtextarea')
      textBox.keypress (e) =>
        e.stopPropagation()
      textBox.keydown (e) =>
        e.stopPropagation()
        code = if e.keyCode then e.keyCode else e.which
        if code == 13
          text = textBox.val()
          textBox.val ""
          e.preventDefault()
          if text != ""
            data = uid: @friend.uid , text: text, channel_name: @channel_name
            QoP.Server.sendRequest('messages',data)
        return
    {% endhighlight%}
    


Eventually, we'll also receive a message. Again, we render the message using handlebars template and append 
the message to the Panel. We also need to scroll the content all the way up to the top, so that we can read 
the message when it arrives.

    
    {% highlight coffeescript%}
    
    messageReceived: (data) =>
      nickname = data.nickname
      message = JST['message'](text: data.text, nickname: nickname)
      @box.find('.chatboxcontent').append(message)
      @box.find('.chatboxcontent').scrollTop 10000000
      return
    {% endhighlight%}
    


That's all Folks!
-----------------

There's a lot more to say, we still have to see how to use Omniauth to authenticate the user, how to use 
DelayedJob to fetch friends data in background, how to setup a custom Sprockets without Rails, and database 
models to hold user and friendship data. But that will be the subject of the next post!

