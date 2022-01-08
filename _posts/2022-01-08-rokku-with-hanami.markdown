---
layout: post
title: "Using Rokku with Hanami"
date: 2022-01-08 16:00:40 +0200
tags: [hanami, rokku, authorization]
categories: Programming
---

In my previous post I've described how to use Tachiban for authentication in a Hanami 1.3 app. This post will be about using my authorization gem [Rokku](https://github.com/sebastjan-hribar/rokku) with Hanami applications.

# Authorization application elements
There are currently four main application elements that drive authorization: users, roles, policies and a custom authorization module.

## 1. Users

### 1.1 Entities
There are only two prerequisites for the user. The user entity must have an attribute of `roles`, which is later on specified in the corresponding policy. Below is an example of a user entity that fulfills prerequisites for both Rokku as well as Tachiban. Notice that `roles` are of type Array. They could also be a String in case users may only have one role.

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
      column :roles, "text[]", null: false
      column :active_status, TrueClass, null: false
    end
  end
end
{% endhighlight %}

I'll be using named roles like `new_user`, `admin` etc. The user assigned role will be compared to the role for a specific policy.

### 1.2 Templates
Here is a shortened example of the new and edit templates for the user. Strings such as labels are represented by symbols to leverage the internationalization.

#### 1.2.1 NEW template
Since the example uses the same controller for admin users and all other users, to edit the template we first need to check if the logged in user has the admin authorization to display the editing roles section for users. Then we make the required checkboxes for assigning roles.

{% highlight ruby %}
if current_user_roles.include?("admin")
  div class: "my-class" do
    div class: "my-form-field" do
      h4 I18n.t :sn_user_attribute_label_roles
    end
    div class: "my-form-field" do
      check_box :roles, name: 'user[roles][]', value: 'new_user', id: 'new_user'
      label I18n.t :sn_list_user_roles_new_user
    end
    div class: "my-form-field" do
      check_box :roles, name: 'user[roles][]', value: 'some-role', id: 'some-role'
      label I18n.t :sn_list_user_roles_some_role
    end
  end
end
{% endhighlight %}

#### 1.2.1 EDIT template
Here we also check for the logged in user authorization. Then we make the required checkboxes for assigning roles with checked options for existing role assignments.

{% highlight ruby %}
if current_user_roles.include?("admin")
  div class: "my-class-row" do
    div class: "my-form-field" do
      h4 I18n.t :sn_user_attribute_label_roles
    end
    div class: "my-form-field" do
      user.roles.include?('new_user')? (check_box :roles, checked: 'checked', name: 'user[roles][]', value: 'new_user') : (check_box :roles, name: 'user[roles][]', value: 'new_user')
      label I18n.t :sn_list_user_roles_new_user
    end
    div class: "my-form-field" do
      user.roles.include?('some_role')? (check_box :roles, checked: 'checked', name: 'user[roles][]', value: 'some_role') : (check_box :roles, name: 'user[roles][]', value: 'some_role')
      label I18n.t :sn_list_user_roles_some_role
    end
  end
end
{% endhighlight %}


## 2. Policies
Each application has its own set of policies. To create a policy for the app `Web` and controller `Notification` we run the following command in the project root folder.

{% highlight ruby %}
rokku -n web -p notification
{% endhighlight %}

Rokku creates the policy file for us. As per instruction we need to uncomment the required roles and add the roles them. In the example below the role `some_user` is authorized for all actions.

{% highlight ruby %}
  module Web
    class NotificationPolicy
      def initialize(roles)
        @user_roles = roles
        # Uncomment the required roles and add the
        # appropriate user role to the @authorized_roles* array.
        @authorized_roles_for_new = ["some_user"]
        @authorized_roles_for_create = ["some_user"]
        @authorized_roles_for_show = ["some_user"]
        @authorized_roles_for_index = ["some_user"]
        @authorized_roles_for_edit = ["some_user"]
        @authorized_roles_for_update = ["some_user"]
        @authorized_roles_for_destroy = ["some_user"]
      end

      def new?
        (@authorized_roles_for_new & @user_roles).any?
      end

      def create?
        (@authorized_roles_for_create & @user_roles).any?
      end

      def show?
        (@authorized_roles_for_show & @user_roles).any?
      end

      def index?
        (@authorized_roles_for_index & @user_roles).any?
      end

      def edit?
        (@authorized_roles_for_edit & @user_roles).any?
      end

      def update?
        (@authorized_roles_for_update & @user_roles).any?
      end

      def destroy?
        (@authorized_roles_for_destroy & @user_roles).any?
      end
    end
  end
{% endhighlight %}


## 3. Prepare the authorization check before call
To autmatically check for authorization for **each request** we can prepare a separate module for that. In such module `Authorization` we then need to make two things.
1. First we need to define all the controllers and their respective singluar forms to match the policy name. 
2. Check if the currently logged in user is authorized. We do this by calling the `authorized?` method of the Rokku. We get the arguments for the method by splitting the controller.


{% highlight ruby %}
module AuthApp
  module Authorization
    def self.included(action)
      action.class_eval do
        before :check_authorization
      end
    end

    private

    def controllers_hash
      { 'Notifications' => 'Notification' }
    end

    def check_authorization
      @user = UserRepository.new.find(session[:current_user])
      app = self.class.to_s.split("::")[0]
      controller = self.class.to_s.split("::")[2]
      action = self.class.to_s.split("::")[3]
      singular_controller = controllers_hash[controller]
      if authorized?(app, singular_controller, action) == false
        flash[:failed_notice] = "You tried to visit an URL you are not authorized for."
        redirect_to '/'
      end
    end
  end
end

{% endhighlight %}


Don't forget to include this module in the `application.rb`.

{% highlight ruby %}
controller.prepare do
  include AuthApp::Authorization
end
{% endhighlight %}

Last but not least, we need to override the `check_authorization` method in all actions where we don't require it.

{% highlight ruby %}
def check_authorization; end
{% endhighlight %}

{% include comments.html %}
