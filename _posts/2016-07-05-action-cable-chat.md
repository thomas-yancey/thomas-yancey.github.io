---
layout: post
title:  "Action Cable Chat -- part 1"
date:   2016-07-05 21:30:06 -0400
categories: jekyll update
---

Rails 5 just came out this past week and I thought what better for a first blog post. I am going to walk through building a simple chat app with action cable. By the end of it we will have an app that allows users to sign up, create private chatrooms and have realtime chat as well as realtime updates of people entering and exiting the room. If you would like to take a look at the finished product you can check it out here -- [Action Cable Chat](http://actionchattin.herokuapp.com/)

<!--more-->

While going through this I am going to skip styling to try and keep the length to a minimum and keep the focus on app development. The first portion of it is just going to be setting up the baseline app without and in part 2 we will jump into action cable. I am going to run through every step of the app so if you want to configure your own setup you might want to just jump to [part 2]({% post_url 2016-07-06-action-cable-chat-two %})

First things first we are going to need to create a new app in Rails 5. If you have yet to upgrade to rails yet go ahead and do so using rails app:update (make sure that you have Ruby 2.2.2 or later). I am going to use postgresql in my app but if you want to use sqlite3 thats fine too. let's also skip turbolinks because it gave me some issues while scoping my chatrooms.

{% highlight powershell %}
rails _5.0.0_ new action-cable-chat -d postgresql --skip-turbolinks
{% endhighlight %}

If you boot up your app you should see the fancy new rails 5 landing page.

On creating the new app you are going to want to uncomment redis in your gemfile. We are eventually going to need it at as our data store for our action cable channels. I am going to also uncomment bcrypt since I am going to build my own user model, but if you want you can use devise to speed things up.

{% highlight powershell %}
bundle install
rails db:create
{% endhighlight %}

Let's go ahead and set up our baseline app. Our database is going to be made up of 4 tables -- users, rooms, messages, and memberships. The memberships table is a join and is where we are going to be able to make private groups.

{% highlight powershell %}
  rails g model User username:string password_digest:string
  rails g model Room name:string
  rails g model Message content:text user:references room:references
  rails g model Membership user:references room:references
{% endhighlight %}

Lets go ahead and add some associations and validations to our models we just generated.

{% highlight ruby %}
class User < ApplicationRecord

  has_secure_password
  has_many :memberships
  has_many :rooms, through: :memberships

  validates_uniqueness_of :username
  validates_presence_of :username
  validates :username, length: {maximum: 16}
end
{% endhighlight %}


{% highlight ruby %}
class Room < ApplicationRecord

  has_many :memberships
  has_many :messages
  has_many :users, through: :memberships

  validates :name, presence: true,
             length: { maximum: 10},
             uniqueness: true
end
{% endhighlight %}

{% highlight ruby %}
class Message < ApplicationRecord

  belongs_to :user
  belongs_to :room

  validates_presence_of :content
  validates_presence_of :user_id
  validates_presence_of :room_id

end
{% endhighlight %}

{% highlight ruby %}
class Membership < ApplicationRecord

  belongs_to :user
  belongs_to :room

  validates_presence_of :room_id
  validates_presence_of :user_id

end
{% endhighlight %}

If you test that out in your console you should be able to create all of the fields. Make sure the relationship is working so that when you call the following you are creating a new Membership.

{% highlight Ruby %}
  User.first.rooms << Room.first
{% endhighlight %}

Now to define some routes including our route path. create a welcome controller with index.

{% highlight powershell %}
rails g controller Welcome index
{% endhighlight %}

we need to setup our routes for users, sessions and root in config/routes.rb

{% highlight ruby %}

  root to: 'welcome#index'.
  resources :users, only: [:new, :index, :create]
  resources :sessions, only: [:new, :destroy, :create]
{% endhighlight %}


Create the corresponding controllers for users and sessions.

{% highlight Ruby %}
class UsersController < ApplicationController

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      session[:user_id] = @user.id
      # Here we are pushing the first room which is
      # going to be our open chat room
      @user.rooms << Room.find(1)
      redirect_to rooms_path
    else
      @errors = @user.errors.full_messages
      render "/users/new"
    end
  end

  private

  def user_params
    params.require(:user).permit(:username, :password)
  end

end
{% endhighlight %}

{% highlight ruby %}
class SessionsController < ApplicationController

  def new
    @user = User.new
  end

  def create
    @user = User.find_by(username: params[:session][:username])
    #utilize bcrypts authenticate method to see if the password is correct
    if @user && @user.authenticate(params[:session][:password])
      session[:user_id] = @user.id
      redirect_to rooms_path
    else
      @errors = @user.nil? ? ["username not found"] : ["username and password do not match"]
      render "/sessions/new"
    end
  end

  def destroy
    session.clear
    redirect_to root_path
  end

end
{% endhighlight %}

You can see we add everyone to a room when the new user is created. The idea of this is to have one open chat room everyne is subscribed to. Add this first room to a seed file and seed the database.

{% highlight ruby %}
#app/db/seeds.rb
Room.create(name: "open_chat")
{% endhighlight %}

We want to be able to access the current user and check whether someone is logged in so lets set up those helpers.

{% highlight ruby %}
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  helper_method :current_user, :logged_in?

  def current_user
    return unless session[:user_id]
    @current_user ||= User.find(session[:user_id])
  end

  def logged_in?
    !!session[:user_id]
  end

end
{% endhighlight %}


This won't work right away because as you can see we set up a redirect to the rooms_path. We want the user to be redirected to their list of rooms which will eventaully be the rooms index. Lets get that all wired up. Go ahead and create the rooms controller and set up your routes.

{% highlight ruby %}
# config/routes.rb
resources :rooms, except: [:update, :edit]

# app/controllers/rooms_controller
class RoomsController < ApplicationController

  def show
    #fill in soon
  end

  def index
    @rooms = current_user.rooms
    @room = Room.new
  end

  def create
    @room = Room.new(room_params)
    if @room.save
      @room.users << current_user
      redirect_to room_memberships_path(@room)
    else
      flash[:error] = @room.errors.full_messages.to_sentence
      redirect_to rooms_path
    end
  end

  private

  def room_params
    params.require(:room).permit(:name)
  end

end

# app/views/rooms/index.html.erb
  HELLO FROM ROOMS INDEX

{% endhighlight %}

At this point we should have a functioning user model. Lets add a basic navbar so that we can test all of this out.

{% highlight erb %}
  #app/view/layouts/application.html.erb right under the body tag
    <nav>
      <ul>
        <% if logged_in? %>
          <li><%= "Welcome back #{current_user.username}" %></li>
          <li><%= link_to "Rooms", rooms_path %></li>
          <li><%= link_to "Logout", session_path(current_user), method: :delete %></li>
        <% else %>
          <li><%= link_to "Login", new_session_path %></li>
          <li><%= link_to "Signup", new_user_path %></li>
        <% end %>
      </ul>
    </nav>

    <% if flash %>
      <% flash.each do |type, message| %>
        <div class="flash">
          <%= message %>
        </div>
      <% end %>
    <% end %>

    <% if @errors %>
      <% @errors.each do |error| %>
        <div class="errors">
          <%= error %>
        </div>
      <% end %>
    <% end %>
{% endhighlight %}

Go ahead and run through login and logouts to see that it is all working appropriately. With that all up and running lets move onto the rooms pages. The idea will be that you can have multiple rooms, and we want to display links to all the show pages for those rooms. So we are going to create two partials that we are going to use inside of the index. One is going to be the form and the other is just the actual room partial.



{% highlight erb %}
#app/views/rooms/_form.html.erb -- new room form partial
<%= form_for room do |f| %>

  <%= f.label :name, "Create new room" %>
  <%= f.text_field :name %>

  <%= f.submit "Create room" %>

<% end %>
{% endhighlight %}

{% highlight erb %}
#app/views/rooms/_room.html.erb -- display room partial
<h3><%= link_to room.name, room_path(room) %></h3>
{% endhighlight %}

{% highlight erb %}
#app/views/rooms/index.html.erb
<h2>Your Chatrooms</h2>
<%= render partial: "/rooms/form",
                     locals: {room: @room} %>

<div class="rooms-list">
  <%= render @rooms %>
</div>
{% endhighlight %}

Alright we are almost ready to start working with action cable. Last things we need to get going are creating new memberships and displaying the rooms. Lets go with creating new memberships first and segway into action cable with showing the rooms. I think it makes the most sense that when a user creates a room he should be directed to a list of users to add to his room. We are going to create a list of current users in the room and a list of users that are not in the room, this is going to be our membership index page. When the creator invites someone off of the list we are going to make an AJAX request, create a membership for that user to the newly created room, remove them from the invite list and add them to members list. Let's build the routes, controller and index page for membership.

{% highlight ruby %}
#config/routes.rb
#lets go ahead and nest these routes with the rooms routes

resources :rooms, except: [:update, :edit] do
  resources :memberships, only: [:index, :create]
end

{% endhighlight %}

{% highlight ruby %}
#app/controllers/memberships_controller.rb
class MembershipsController < ApplicationController

  def index
    @room = Room.find(params[:room_id])
    redirect_with_flash unless member_of_group
    @memberships = Membership.where(room_id: params[:room_id])
    @users = User.where.not(id: @memberships.pluck(:user_id))
  end

end
{% endhighlight %}

 So you can see that we make a redirect unless member_of_group. Basically we are going to let other members invite to this group after they have been added, but we only want users that are members of the group to be able to add users. We are going to need to create that helper. Let's also go ahead and add a verify logged in method to redirect anyone that isn't signed in. We can attach it as a before action to all of the controllers except users and sessions.

{% highlight ruby %}
#app/controllers/application_controller
  helper_method :current_user, :logged_in?, :member_of_group,
   :verified_log_in, :redirect_with_flash

  def member_of_group
    !!@room.memberships.find_by(user_id: current_user.id)
  end

  def redirect_with_flash
    flash[:notice] = "You are not a member of that room"
    redirect_to root_path
  end

  def verify_logged_in
    unless logged_in?
      flash[:notice] = "You must be logged in to view that page"
      redirect_to root_path
    end
  end

{% endhighlight %}

{% highlight ruby %}
#Add the following to the top of the memberships, messages, and rooms controller
  before_action :verify_logged_in
{% endhighlight %}

And finally lets create the index view

{% highlight erb %}
#app/views/memberships/index.html.erb
<h3><%= link_to "go to #{@room.name}",room_path(@room) %></h3>
<h2 >Add members to <%=@room.name%></h2>
<h3>Members</h3>

<ul id="member-list">
  <% @memberships.each do |member| %>
    <li><%= member.name %></li>
  <% end %>
</ul>

<ul id="#add-users">
<% @users.each do |user| %>
  <li>
    <%= user.username %>
    <input type="checkbox" id="<%= "#{user.id}-add-member" %>"/>
  </li>
<% end %>
</ul>
{% endhighlight %}

Also a quick method on our membership model to make accessing the member name a little cleaner.

{% highlight ruby %}
# app models/membership.rb
  def name
    User.find(self.user_id).username
  end
{% endhighlight %}

Now what you <em>should</em> do for the index is break those two list iterations out into helpers but for the purpose of brevity we are just going to keep moving. The idea here is when a member user is on this page they can click the checkbox to add a user to the page. Time to create our create action for members. We are only going to get XHR requests for this app and not worry about older compatibility or error handling.

{% highlight ruby %}
#app/controllers/memberships_controller.rb
  def create
    @room = Room.find(params[:room_id])
    redirect_with_flash unless member_of_group

    user = User.find(params[:user_id])
    membership = Membership.new(user_id: user.id,
                                room_id: @room.id)
    if membership.save
      json_hash = {username: user.username, id: user.id}
      render text: JSON.generate(json_hash)
    else
      #error handling
    end
  end
{% endhighlight %}

Now lets AJAX this puppy! on click of the div we want to create a membership for the clicked user to the current room. We access the current room through the nested params and we access the user id from the string we use for the input id.

{% highlight javascript %}
// app/assets/javascripts/main.js

$( document ).ready(function() {
  $('#add-users input').on ('click', function(){
    var userId = $(this).attr("id").split("-")[0];
    var data = {user_id: userId};
    $.ajax({
      url: this.action,
      data: data,
      method: 'post'
    }).done(function(res){
      var userData = JSON.parse(res);
      $('#' + userData.id + "-add-member").parent().remove();
      $('#member-list').append(memberAppendBuilder(userData));
    });
  });
});

var memberAppendBuilder = function(userData){
   return "<li>" + userData.username + "</li>";
};

{% endhighlight %}

I want to add one validation here to our Membership. We should validate that no duplicate records are created so that if some issue comes up on the return value of the AJAX response the user doesn't just keep clicking and making duplicates in our database.

{% highlight ruby %}
#app/models/membership.rb
validates_uniqueness_of :user_id, :scope => :room_id
{% endhighlight %}

Last but not least we need to create our rooms show page. After we have this up and running we are ready to integrate our WebSockets!!! So for this bit we want to have our rooms first render all of the messages, we are gonna limit it to 1000 cause this app is gonna take off! Let's go ahead and fill out the show action in rooms and create the view.

{% highlight ruby %}
#app/controllers/rooms_controller.rb
  def show
    @room = Room.find(params[:id])
    redirect_with_flash unless member_of_group

    @messages = @room.messages.order(id: :desc).limit(500).reverse
    @message = Message.new
    @users = @room.users
  end
{% endhighlight %}

{% highlight erb %}
# app/views/show.html.erb
<h3><%= @room.name %></h3>

<ul id="users-list">
  <% @users.each do |user| %>
      <li>
        <%= user.username %>
        <%= image_tag "green_dot.png", id: "#{user.id}-status"%>
      </li>
  <% end %>
</ul>

<%= link_to "invite people", room_memberships_path(@room) %>
<%= render @messages %>

<%= render partial: "/messages/form", locals: {room: @room, message: @message} %>

{% endhighlight %}

We are going to use that green-dot.png to show when users enter and exit the room eventually. If you are following along you can grab the image from this [repo]({{ "https://github.com/thomas-yancey/action-cable-chat-template" }}).

Now we need to put the form  and message partials we called above in the messages view folder. Lets create that controller and then make the partials.

{% highlight erb %}
#app/views/messages/_form.html.erb

<%= form_for message do |f| %>

  <%= f.hidden_field :room_id, value: room.id %>

  <%= f.label :content, "enter message here" %>
  <%= f.text_field :content, autocomplete: :off %>

<% end %>
{% endhighlight %}

{% highlight erb %}
#app/views/messages/_message.html.erb

<%= form_for message do |f| %>

  <%= f.hidden_field :room_id, value: room.id %>

  <%= f.label :content, "enter message here" %>
  <%= f.text_field :content, autocomplete: :off %>

<% end %>
{% endhighlight %}

The form partial still is going to give us trouble until we set up that create route for messages. Let's round out the messages controller and we can finally move on. Let's set up the create route.

{% highlight ruby %}
#config/routes.rb

resources :messages, only: [:create]
{% endhighlight %}

and set up the message create. Just to test it out we will have it redirect. On the second part of this post we will wire it up through action cable.

{% highlight ruby %}
#app/controllers/messages_controller.rb
class MessagesController < ApplicationController

  def create
    @message = Message.new(message_params)
    @message.save
    redirect_to :back
  end

  private

  def message_params
    params.require(:message).permit(:room_id, :content).merge(user_id: current_user.id)
  end

end
{% endhighlight %}

Alright! We made it through, currently we have a pretty basic outline of an app. You can login, logout, create chatrooms, add members, and post messages, Next post we are going make this chat realtime!

## Go through all of your before action for validations

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
