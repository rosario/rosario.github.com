---
layout: post
title: Call Forwarding with Twilio and Sinatra
image: '/images/telefono.png'

---

Say you want to buy a virtual phone number, and you'd like to redirect calls to your mobile number. There could 
be different reasons why you'd want to have another phone number, without having to carry a physical phone.

<div class="center">
  <img src="/images/telefono.png" alt="phone">
</div>

A typical case is when you submit your CV to websites. Instead of giving away your mobile number, you
could buy a number from [Twilio](http://twilio.com "Twilio"), create a micro application hosted for free on
heroku, and let Twilio redirect those calls to your mobile number. 

You talkin' to ME ?!?
---------------------

Twilio uses [TWIML](http://www.twilio.com/docs/api/twiml "TWIML"), a markup language that describes operations like **DIAL** or **REJECT**. 

When somebody calls our virtual phone number, Twilio will make a _POST_ request to our server. We return some TWIML 
instructing Twilio to forward that call to our personal (mobile or landline) phone:

      post '/:twilio_number' do
        if @account and @account.forward_number
          # Dial phone number
          builder do |xml|
            xml.instruct!
            xml.Response do
                xml.Dial @account.forward_number,
                  :action =>"/call_status/#{params[:twilio_number]}"
            end  
          end
        else
          # Reject the call, the number is not active
          builder do |xml|
            xml.Response do
              xml.Reject
            end
          end
        end
      end

Note, this **POST** request is made by Twilio to our server. We need to configure our Twilio Account and 
associate an _URL_ with our virtual phone number.

The following **action** is executed when the phone hangs up, we just instruct Twilio to close the call:

      post '/call_status/:twilio_number' do
        builder do |xml|
          xml.instruct!
          xml.Response do
            xml.Reject
          end
        end
      end
    


Logging
-------

We can use a database to log calls too, and it's quite easy to setup with **Datamapper** and Sinatra.
Here's a simple database model:


      class Account
        include DataMapper::Resource
        property :id,             Serial
        property :email,          String
        property :twilio_number,  String
        property :forward_number, String
        property :created_at,     DateTime
        has n, :calls
      end


      class Call
        include DataMapper::Resource
        property :id,                 Serial
        property :account_sid,        String
        property :to_zip,             String
        property :to_city,            String
        property :to_state,           String
        property :to_country,         String
        property :called_zip,         String
        property :called_city,        String
        property :called_country,     String
        property :called_state,       String
        property :call_status,        String
        property :call_sid,           String
        property :called,             String
        property :from,               String
        property :to,                 String
        property :caller,             String
        property :dial_call_duration, Integer, :default => 0
        property :dial_call_status,   String
        property :dial_call_sid,      String
        property :direction,          String
        property :api_version,        String
        belongs_to :account
      end

Twilio provides the details of who's calling, from where and other useful infos. We can grab this informations
for each request, with a **before** filter in Sinatra:

      before do
        if params["To"] and params["AccountSid"] and params["CallStatus"]
          call = Call.new({    
            :account_sid =>   params["AccountSid"],
            :call_status =>   params["CallStatus"],
            :to_zip =>        params["ToZip"],
            :to_city =>       params["ToCity"],
            :to_state =>      params["ToState"],
            :called =>        params["Called"],
            :to =>            params["To"],
            :to_country =>    params["ToCountry"],
            :called_zip =>    params["CalledZip"],
            :direction =>     params["Direction"],
            :api_version =>   params["ApiVersion"],
            :caller =>        params["Caller"],
            :called_city =>   params["CalledCity"],
            :called_country =>      params["CalledCountry"],
            :dial_call_duration =>  params["DialCallDuration"].to_i || 0,
            :dial_call_status =>    params["DialCallStatus"] || "",
            :dial_call_sid =>       params["DialCallSid"] || "",
            :call_sid =>      params["CallSid"],
            :called_state =>  params["CalledState"],
            :from =>          params["From"]
          })
          account = Account.first(:twilio_number=>call.to)
          account.calls << call
          account.save
          call.save
          @account = account
        end
      end
      
      
Conclusion
----------

It's easy, really easy to setup a simple call forwarding service with Twilio. When we get tired of receiving calls
we can always disable the virtual phone number, hassle free. 

Source code can be found here: [callforwarding](https://github.com/rosario/callforwarding "callforwarding")