---
layout: post
title: "Using redis for session store in a Hanami application"
date: 2019-07-23 17:00:00 +0200
---

To be able to list logged in users in my Hanami application I've switched from cookies
to redis for session store. The setup is quite brief thanks to the redis-rack gem.

## 1. Gemfile

Add the gem to the Gemfile:

{% highlight ruby %}
gem 'redis-rack'
{% endhighlight %}


## 2. application.rb

Require redis:
{% highlight ruby %}
require 'rack/session/redis'
{% endhighlight %}

Switch from cookies to redis:
{% highlight ruby %}
#sessions :cookie, secret: ENV['WEB_SESSIONS_SECRET']
sessions :redis, secret: ENV['WEB_SESSIONS_SECRET']
{% endhighlight %}

## 3. Admin panel controller action

In my simple example below the code returns an array of logged-in user IDs to then query from the DB.

{% highlight ruby %}
def call(params)
  store = Redis::Store.new
  session_keys = store.keys
  
  users = []
  
  session_keys.each do |s_key| 
    s_value = store.get(s_key)
    users << s_value["current_user"]
  end
end
{% endhighlight %}


The `s_key` in returns the rack session ID and the `s_value` returns the everything stored in the session:

{% highlight ruby %}
s_key: rack:session:1sdrf724945701f13d0a5hju8ffb0078572fser42137948sdfr0d1fhju89bae4a
s_value: {"_csrf_token"=>"9sfg5461cfsdgh7285e16616c974we45t0db5c73dcfc1312b3c9ab91937b809c2", "LOCK"=>"Off", "current_user_class"=>Teacher, "session_start_time"=>2019-07-25 17:26:19 +0200, "__last_request_id"=>"sde458c7454745tg6d8a964b223456f2", "current_user"=>1}
{% endhighlight %}

#### Credits
Thanks to Luca Guidi, the creator of the Hanami framework.
