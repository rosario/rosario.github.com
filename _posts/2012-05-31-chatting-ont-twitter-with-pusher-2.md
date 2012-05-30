---
layout: post
title: Chatting on Twitter (2/2)
image: '/images/screenshot.png'
comments_url: "http://news.ycombinator.com/item?id=3998828"
excerpt: |-
    Can we embed a chat widget on <strong>Twitter</strong>? Yes, with few tricks and <strong>Pusher</strong> we can create a widget and chat with our Twitter friends.
    
---


### Can we embed a chat widget on **Twitter**? Yes, with few tricks and **Pusher** we can create a widget and chat with our Twitter friends.

![qop chat](/images/screenshot.png "qop chat screenshot")


Overview
--------


In the previous post, we have seen how to embed the widget using Coffescript. 
We mainly covered events sent from and to the server, and how to render messages 
and chat box. In this post we will cover more database structure, how to setup Sprocket 
without Rails, and finally how to use Delayed Job with Sinatra to retrieve the user's friends 
in background.



Relationship status: It's not complicated
-----------------------------------------

We use [Datamapper]('http://datamapper.org/' "Datamapper")
to write database models. It's relatively easy to use and we don't have to worry about 
migrations (and that's ok given this simplicity of this project). Documentation is really 
good, we reuse part of _Self referential many to many relationships_ section in 
[Associations](http://datamapper.org/docs/associations.html "Associations")
and we add few utility methods.

    {% highlight ruby%}
    
    class User
      include DataMapper::Resource

      property :id,         Serial
      property :name,       String
      property :uid,        String, :required => true
      property :name,       String
      property :nickname,   String
      property :created_at, DateTime
      property :online,     Boolean, :default  => false

      has n, :friendships, :child_key => [ :source_id ]
      has n, :friends, self, :through => :friendships, :via => :target

      has n, :inverse_friendships, 'Friendship', :child_key =>[:target_id]
      has n, :inverse_friends, self, :through =>:inverse_friendships, :via =>:source
      def online!
        self.online = true
        self.save
      end

      def friend?(friend)
        friends.first(:id =>friend.id) or inverse_friends.first(:id=>friend.id)
      end


      def offline!
        self.online = false
        self.save
      end

      def online_friends
        fs = friends.all(:online=>true)
        ls = inverse_friends.all(:online=>true)
        fs + ls
      end

    end

    class Friendship
      include DataMapper::Resource
      belongs_to :source, 'User', :key => true
      belongs_to :target, 'User', :key => true
    end
    {% endhightlight%}

We have simply added **online** and **offline** methods, a method to get online friends and 
the _inverse\_friendship_ relation. If user _A_ is friend with user _B_, then it means _B_ is also 
friend with _A_. We store this relation only once in the **Friendship** table, but we use the inverse 
friendship relation to change between _source_ and _target_.

Sprockets without Rails
-----------------------

Sprockets is a utility to compile coffescript/javascript and stylesheets. It's very useful when we write 
lots of javascript and we need a way to package all the files in a single application.js. It also 
supports _ECO_ templates and automatically creates a global variable to store our javascript templates.

Blatantly taken from [sprockets without rails](http://www.simonecarletti.com/blog/2011/09/using-sprockets-without-a-railsrack-project/ "sprockets without rails")
courtesy of [Simone Carletti](http://www.simonecarletti.com/ "Simone Carletti"), there is an effective way to setup 
Sprockets for a _Rack_ application. The following code goes in a **Rakefile.rb**.

    
    {% highlight ruby %}
    
    require 'rubygems'
    require 'bundler'
    require 'pathname'
    require 'logger'
    require 'fileutils'
    require 'data_mapper'

    Bundler.require

    ROOT        = Pathname(File.dirname(__FILE__))
    LOGGER      = Logger.new(STDOUT)
    BUNDLES     = %w( application.css application.js )
    BUILD_DIR   = ROOT.join("public")
    SOURCE_DIR  = ROOT.join("assets")



    task :compile do
      sprockets = Sprockets::Environment.new(ROOT) do |env|
        env.logger = LOGGER
      end

      sprockets.append_path(SOURCE_DIR.join('javascripts').to_s)
      sprockets.append_path(SOURCE_DIR.join('templates').to_s)
      sprockets.append_path(SOURCE_DIR.join('stylesheets').to_s)

      BUNDLES.each do |bundle|
        assets = sprockets.find_asset(bundle)
        prefix, basename = assets.pathname.to_s.split('/')[-2..-1]
        FileUtils.mkpath BUILD_DIR.join(prefix)

        assets.to_a.each do |asset|
          # strip filename.css.foo.bar.css multiple extensions
          realname = asset.pathname.basename.to_s.split(".")[0..1].join(".")
          asset.write_to(BUILD_DIR.join(prefix, realname))
        end

        assets.write_to(BUILD_DIR.join(prefix, basename))
      end
    end
    {% endhightlight %}


It is also useful to automatically save and compile javascript files without manually running rake tasks. 
To do so, we setup a **watchr** file, and install the gem [watchr](https://github.com/mynyml/watchr "watchr").

    {% highlight ruby %}

    def run_sprockets
      print "\nCompiling... "
      system('rake compile')
      print "done.\n"
    end

    Signal.trap('INT') { abort("\n") }
    watch( 'assets/javascripts/*') { run_sprockets}
    watch( 'assets/templates/*') { run_sprockets}    
    {% endhightlight %}
    

Omniauth for Twitter using Sinatra
----------------------------------

[Omniauth](https://github.com/intridea/omniauth "Omniauth") makes it very easy to authenticate with Twitter. 
We just need to provide a callback method, and get the user informations.


    {% highlight ruby %}
    
    get '/auth/twitter/callback' do
    
      auth = request.env["omniauth.auth"]
      user = User.first_or_create({ :uid => auth["uid"]}, {
        :uid => auth["uid"],
        :nickname => auth["info"]["nickname"],
        :name => auth["info"]["name"],
        :created_at => Time.now })

      if user.nickname.nil? or user.nickname.empty?
        user.nickname = auth["info"]["nickname"]
        user.save
      end

      if user and user.nickname and user.friends.empty?
        job = LookupFriends.new(user.nickname)
        Delayed::Job.enqueue job
      end
      
      session[:user_id] = user.id
      redirect 'https://twitter.com'
      
    end
    {% endhightlight %}


Sinatra and Delayed Job
-----------------------

In the previous code snippet we use _Delayed Job_ to fetch out first friends list. To setup 
[delayed job](https://github.com/collectiveidea/delayed_job "delayed job") 
with Sinatra is a piece of cake. We need to define few _tasks_ in our _Rakefile.rb_. Note that the following 
task are useful when we deploy the code to [heroku](http://www.heroku.com "heroku").

    {% highlight ruby %}
    
    task :environment do
      require 'delayed_job'
      require 'delayed_job_data_mapper'
      require './models'
      require './job'
      DataMapper.setup(:default, ENV['DATABASE_URL'] || "sqlite3://#{Dir.pwd}/db.sqlite3")
      DataMapper.finalize
    end

    namespace :jobs do
      desc "Clear the delayed_job queue."
      task :clear => :environment do
        Delayed::Job.delete_all
      end

      desc "Start a delayed_job worker."
      task :work => :environment do
        Delayed::Worker.new(:min_priority => ENV['MIN_PRIORITY'], :max_priority => ENV['MAX_PRIORITY']).start
      end
    end
    {% endhightlight %}



Delayed Job is now ready to work, the _job_ we use is the following:


    {% highlight ruby %}
    
    class LookupFriends < Struct.new(:nickname)
      def perform
        friends = Twitter.friend_ids(nickname.to_s)
        user = User.first(:nickname => nickname)
        if friends.ids
          friends.ids.each do |id|
            friend = User.first_or_create(:uid=> id)
            user.friends << friend unless user.friends.first(:uid=> id)
          end
          user.save
        end
      end
    end
    {% endhightlight %}



Conclusions
-----------





