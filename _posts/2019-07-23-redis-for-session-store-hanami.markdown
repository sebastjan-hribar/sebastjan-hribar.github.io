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

Switch from
{% highlight ruby %}
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
