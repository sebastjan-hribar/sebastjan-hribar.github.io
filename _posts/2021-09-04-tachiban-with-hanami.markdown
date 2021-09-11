---
layout: post
title: "Using Tachiban with Hanami"
date: 2021-09-03 22:25:40 +0200
tags: [hanami, tachiban, authentication]
categories: Programming
---

In this post I'll describe how I use the authentication gem [Tachiban](https://github.com/sebastjan-hribar/tachiban) in my Hanami 1.3 applications.
I use a separate Hanami application to handle authentication and authorization. I'll focus on the authorization ([Rokku](https://github.com/sebastjan-hribar/rokku)) in a separate post.

# Authentication application elements
There are currently five main application elements that drive authentication: users, user sessions, dashboard, password reset/update and setting Tachiban defaults. I use one separate module for setting the Tachiban defaults, while I override certain Tachiban methods where appropriate.

## 1. Users

### 1.1 Entities
The user attributes are defined as follows to make use of Tachiban as per prerequisites defined [here](https://github.com/sebastjan-hribar/tachiban#prerequisites).

{% highlight ruby %}
Hanami::Model.migration do
  change do
    create_table :users do
      primary_key :id

      column :created_at, DateTime, null: false
      column :updated_at, DateTime, null: false

      column :name, String, null: false
      column :surname, String, null: false
      column :username, String, null: false
      column :email, String, null: false, unique: true
      column :password_reset_sent_at, DateTime
      column :hashed_pass, String, null: false
      column :token, String
      column :role, String, null: false
      column :active_status, TrueClass, null: false
    end
  end
end
{% endhighlight %}

### 1.2 Controllers
The most relevant action for Tachiban is `Create`. The `translate_err_mess` method in the sample code below is a custom method for translating error messages. I need this to adjust translations for Slovenian, for example.

The most important thing is to setup a hashed password to be saved in the database.

{% highlight ruby %}
module Users
  class Create
    include AuthApp::Action

    params do
      configure do
        config.messages = :i18n
      end
      required(:user).schema do
        required(:name).filled(:str?, min_size?: 3)
        required(:surname).filled(:str?, min_size?: 3)
        required(:username).filled(:str?, min_size?: 3)
        required(:email).filled(format?: /\A[^@]*?@[\w\-\.]*?\.\w*\z/)
        required(:password).filled(size?: 8..32)
      end
    end

    def call(params)
      if params.valid?
        begin
          password = params[:user][:password]
          params[:user][:hashed_pass] = hashed_password(password)
          params[:user][:role] = 'new_user'
          params[:user][:active_status] = true
          UserRepository.new.create(params[:user])
          redirect_to routes.users_path
        rescue StandardError => e
          @standard_error = e
          self.status = 422
          redirect_to routes.new_user_path
        end
      else
        @params_errors = translate_err_mess(params.errors)
        self.status = 422
        redirect_to routes.new_user_path
      end
    end
  end
end
{% endhighlight %}


### 1.3 Routes

{% highlight ruby %}
resources :users
{% endhighlight %}


## 2. User sessions

### 2.1 Entities
There are no entities required for user sessions, but only controllers and actions.

### 2.2 Controllers
 The three actions for the user sessions controller are:
* `New`
* `Create` and
* `Destroy`.

#### 2.2.1 New
In the `New`action I first set the current user to nil and also override the methods that check for the logged in user and handle session. This is needed in order to prevent an infinite loop of checking and redirecting.

{% highlight ruby %}
module AuthApp
  module Controllers
    module UserSessions
      class New
        include AuthApp::Action

        def call(params)
          session[:current_user] = nil
        end

        private

        def check_for_logged_in_user; end

        def handle_session; end
      end
    end
  end
end
{% endhighlight %}

#### 2.2.2 Create
The `Create` action finds the user trying to log in and if they are authenticated they are logged in. Otherwise the `logout`method is called. The `check_for_logged_in_user`and `handle_session` are overridden again.

{% highlight ruby %}
module AuthApp
  module Controllers
    module UserSessions
      class Create
        include AuthApp::Action

        params do
          required(:user_session).schema do
            required(:email).filled(format?: /\A[^@]*?@[\w\-\.]*?\.\w*\z/)
            required(:password).filled(size?: 8..64)
          end
        end

        def call(params)
          @user = UserRepository.new.find_by_email(params[:user_session][:email])
          if authenticated?(params[:user_session][:password])
            login
          else
            logout
          end
        end

        private

        def check_for_logged_in_user; end

        def handle_session; end
      end
    end
  end
end
{% endhighlight %}


#### 2.2.3 Destroy
Lastly, the `Destroy`action logs the user out. This acion also requires bypassing the Tachiban's checking methods.

{% highlight ruby %}
module AuthApp
  module Controllers
    module UserSessions
      class Destroy
        include AuthApp::Action

        def call(params)
          logout
        end

        private
        def check_for_logged_in_user; end

        def handle_session; end
      end
    end
  end
end
{% endhighlight %}


### 2.3 Routes
{% highlight ruby %}
resources :user_sessions, only: [:create]
get '/login', to: 'user_sessions#new'
get '/logout', to: 'user_sessions#destroy'
{% endhighlight %}


## 3. Dashboard
The dashboard controller is basically a home page for the app and thus not needed for the Tachibam implementation. I include it here to better ilustrate the entire app.

### Routes
{% highlight ruby %}
root to: 'dashboard#index'
get '/dashboard', to: 'dashboard#index'
{% endhighlight %}


## 4. Password reset/update

### 4.1 Entities
There are no entities required for user sessions, but only controllers and actions. For both I use the EDIT/UPDATE actions. The templates are not included here.

### 4.2 Controllers
There are two set of actions for this functionality: one for requesting the password reset (or to handle a forgotten password) and the other for updating the password.

#### 4.2.1 Password reset UPDATE action

{% highlight ruby %}

module AuthApp
  module Controllers
    module Passwordreset
      class Update
        include AuthApp::Action

    params do
      #specify and validate params
    end

    def call(params)
      if params.valid?
        email = params[:passwordreset][:email]
        user_repo = UserRepository.new
        user = user_repo.new.find_by_email(email)

        # Generate the token and update user with the time of the password reset link
        # and the generated token.
        token = SecureRandom.urlsafe_base64
        password_reset_sent_time = Time.now
        user_repo.update(user.id, password_reset_token: token, password_reset_sent_at: password_reset_sent_time)

        # Set the reset e-mail title and body
        title = "Password reset"
        body = "http://localhost:2300/authapppasswordupdate/#{token}"

        # Send the reset email
        Mailers::Passwordreset.deliver(mail_title: title, mail_body: body, user_email: email)

        flash[:success_notice] = "Password reset link sent."
        redirect_to '/'
      else
        #
        #
        #
      end
    end

    private
    def check_for_logged_in_user; end

    def handle_session; end
  end
end
{% endhighlight %}


#### 4.2.2 Password update UPDATE action

{% highlight ruby %}
module AuthApp
  module Controllers
    module Passwordupdate
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

        # Find user by token.
        user_repo = UserRepository.new
        user = user_repo.user_by_token(token)

        # Check for reset link validity
        if !password_reset_url_valid?(7200)
          flash[:failed_notice] = "Reset link validity (2 hours) has expired."
          redirect_to "/"
        else
          password_salt = BCrypt::Engine.generate_salt
          password_hash = BCrypt::Engine.hash_secret(@new_pass, password_salt)

          user_repo.update(user.id, password_hash: password_hash, password_salt: password_salt)

          flash[:success_notice] = "The password was reset."
          redirect_to "/"
        end
      else
        #
      end
    end
  
  private
  def check_for_logged_in_user; end

  def handle_session; end
  end
end
{% endhighlight %}


### 4.3 Mailers
{% highlight ruby %}
class Mailers::Passwordreset
  include Hanami::Mailer

  from    'my-app@some-domain.com'
  to      :recipient
  subject :mail_title


  private

  def emailbody
    mail_body
  end

  def recipient
    user_email
  end

end
{% endhighlight %}

Also don't forget to setup the delivery configuration in the `environment.rb`.

### 4.4 Routes
In order for the code below to work these routes have to be defined:

{% highlight ruby %}
resource :passwordreset, only: [:edit, :update]
get '/passwordupdate/:token', token: /([^\/]+)/, to: 'passwordupdate#edit'
patch '/passwordupdate/:token', token: /([^\/]+)/, to: 'passwordupdate#update'
{% endhighlight %}


## 5. Setting Tachiban defaults
In `Authentication` module I do the following.
1. Specify Tachiban's methods that I want to call in the before block for every action.
2. Set default urls for my application.

{% highlight ruby %}
module AuthApp
  module Authentication
    def self.included(action)
      action.class_eval do
        before :handle_session_redirect_url, :logout_redirect_url, :login_redirect_url, :check_for_logged_in_user, :handle_session
      end
    end

    private

    def handle_session_redirect_url
      @redirect_url = '/authapp/login'
    end

    def logout_redirect_url
      @logout_redirect_url = '/authapp/login'
    end

    def login_redirect_url
      @login_redirect_url = '/user_home'
    end

  end
end
{% endhighlight %}

Don't forget to include this module in the `application.rb`.

{% highlight ruby %}
#
#
controller.prepare do
  include AuthApp::Authentication
  # include MyAuthentication # included in all the actions
  # before :authenticate!    # run an authentication before callback
end
#
#
{% endhighlight %}


{% include comments.html %}