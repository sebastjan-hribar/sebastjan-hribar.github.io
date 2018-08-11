---
layout: post
title:  "Authentication for Hanami applications"
date:   2018-07-20 09:45:40 +0200
---

I wanted to learn about the workings of an authenitaction system for a web application and I decided to make my own for a Hanami based web application I was working on at the time.

I've exctracted the code and system to a separate gem [Tachiban](https://github.com/sebastjan-hribar/tachiban).

Below I'll try to retrack the steps in the process of setting up my own the auth system.

## 1. Preparing the user entity

# 1.1 The entity

{% highlight ruby %}
class User
  include Hanami::Entity

  attributes :name, :surname, :email, :password, :password_hash, :password_salt,
  :password_confirmation, :password_reset_token, :password_reset_sent_at, :active_status,
  :username

end
{% endhighlight %}

# 1.2 The mapping

{% highlight ruby %}
collection :users do
  entity     User
  repository UserRepository

  attribute :id,                      Integer
  attribute :name,                    String
  attribute :surname,                 String
  attribute :email,                   String
  attribute :password_hash,           String
  attribute :password_salt,           String
  attribute :password_reset_token,    String
  attribute :password_reset_sent_at,  DateTime
  attribute :active_status,           String
  attribute :username,                String
end
{% endhighlight %}


# 1.3 The create action for the user entity

I use [[bcrypt|https://rubygems.org/gems/bcrypt/versions/3.1.11]] gem for password encryption, so prior to user creation we need to setup the password salt and hash.

{% highlight ruby %}
password_salt = BCrypt::Engine.generate_salt
password_hash = BCrypt::Engine.hash_secret(params[:user][:password], password_salt)
{% endhighlight %}

Then we create the user:

{% highlight ruby %}
@user = UserRepository.create(User.new(name: name, surname: surname, email: email, password_hash: password_hash, password_salt: password_salt, active_status: active_status, username: username))
{% endhighlight %}

## 2. Creating the session and logging the user in

{% highlight ruby %}
module Web::Controllers::UserSessions
  class Create
    include Web::Action

    params do
      ## specify and validate params
      end
    end

    def call(params)
      if params.valid?
        email = params[:user_session][:email]
        password = params[:user_session][:password]

        # User is looked up in the repository by email
        @user = UserRepository.user_with_email(email)

        # Email and password (ad hoc encrypted) are 
        #compared with those in the database and
        #corresponding control flow is in place.
        if !@user
          session[:current_user] = nil
          flash[:failed_notice] = "The user account
          doesn't exist."
          redirect_to '/'
        else
          if @user && @user.password_hash == 
          BCrypt::Engine.hash_secret(password,
         @user.password_salt)
            session[:current_user] = @user
            flash[:success_notice] = "You are now logged in."
            redirect_to '/user_home'
          else
            session[:current_user] = nil
            flash[:failed_notice] = "Your log in data
             was incorrect."
            redirect_to '/'
          end
        end
      else
        #
        #
      end
    end
  end
end
{% endhighlight %}


## 3. Authentication methods

I've setup some methods in an authentication module that can be run before actions. The code below is setup for development mode so the test clauses are still there.


{% highlight ruby %}
    def check_for_user_class
      unless ENV['HANAMI_ENV'] == 'test'
        halt 401 unless session[:current_user].class == User
      end
    end

    def check_for_admin_role
      unless ENV['HANAMI_ENV'] == 'test'
        halt 401 unless session[:current_user].role == "administrator"
      end
    end

    def check_for_signed_in_user
      unless ENV['HANAMI_ENV'] == 'test'
        halt 401 unless session[:current_user]
      end
    end
{% endhighlight %}


## 4. Logout

{% highlight ruby %}
module Web::Controllers::Logout
  class Index
    include Web::Action

    # Logs the current user out by setting its session value to nil.
    def call(_)
      session[:current_user] = nil
      flash[:success_notice] = "You've been logged out."
      redirect_to "/"
    end
  end
end
{% endhighlight %}


## 5. Password reset

# 5.1 Password reset link request action

In order for the code below to work these routes have to be defined:


{% highlight ruby %}
get '/passwordupdate/:token', token: /([^\/]+)/, to: 'passwordupdate#edit'

patch '/passwordupdate/:token', token: /([^\/]+)/, to: 'passwordupdate#update'

{% endhighlight %}


{% highlight ruby %}
module Web::Controllers::Passwordreset
  class Update
    include Web::Action

    params do
      #specify and validate params
    end

    def call(params)
      if params.valid?
        # Sets variables from param values
        email = params[:passwordreset][:email]
                
        user = UserRepository.user_with_email(email)
        
        
        # Generate the token and update user with the time of the password reset link
        # and the generated token.
        token = SecureRandom.urlsafe_base64
        password_reset_sent_time = Time.now
        user.update(password_reset_token: token, password_reset_sent_at: password_reset_sent_time)
        user = repository.update(user)

        # Set the reset e-mail tiel and body
        title = "Ponastavitev gesla"
        body = "http://localhost:2300/passwordupdate/#{token}"
        
        # Send the reset email
        Mailers::Passwordreset.deliver(mail_title: title, mail_body: body, user_email: email)
        
        
        flash[:success_notice] = "Password reset link sent."
        redirect_to '/'

        
      else
        flash[:failed_notice] = "The e-mail wasn't found."
        redirect_to 'passwordreset/edit'
      end
    end
  end
end
{% endhighlight %}


# 5.2 Password update action

{% highlight ruby %}
module Web::Controllers::Passwordupdate
  class Update
    include Web::Action

    params do
      param :token, presence: true
      param :passwordupdate do
        param :new_password, presence: true
        param :repeat_password, presence: true
      end
    end

    def call(params)
      if params.valid?
        @new_pass = params[:passwordupdate][:new_password]
        token = params[:token]
        
        # Find user by token, looking at both repositories.     
        @user = UserRepository.user_by_token(token)
        @repository = UserRepository
 
        # Check for reset link validity
        if Time.now > @user.password_reset_sent_at.to_time + 7200
          flash[:failed_notice] = "Reset link validity (2 hours) has expired."
          redirect_to "/"
        else
          password_salt = BCrypt::Engine.generate_salt
          password_hash = BCrypt::Engine.hash_secret(@new_pass, password_salt)
          
          @user.update(password_hash: password_hash, password_salt: password_salt)
          @user = @repository.update(@user)     
   
          flash[:success_notice] = "The password was reset."
          redirect_to "/"
        end
      else
        #
      end
    end
  end
end
{% endhighlight %}

{% include comments.html %}
