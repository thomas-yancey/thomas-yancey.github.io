---
layout: post
title:  "Action Cable Chat -- part 2"
date:   2016-07-06 21:30:06 -0400
categories: jekyll update
---


Last Post we went through and built out the chat app without any of the realtime features. If you want to work along you can fork the repo from here [action-cable-chat-example]({{https://github.com/thomas-yancey/action-cable-chat-template}}).

<!--more-->

Alright so one of the first things we want to do with our action cable is set up the connection between the client and the server. When you create your rails 5 app a lot of this is already given to you. We have to unomment out the bit in config/routes to mount the action cable.

{% highlight ruby %}
# config/routes.rb
mount ActionCable.server => '/cable'
{% endhighlight %}

Make sure that your application head you include the action_cable_meta_tag

{% highlight erb %}
# app/views/layouts/application.html.erb
<head>
  ...
  <%= action_cable_meta_tag %>
  ...
</head>
{% endhighlight %}

And for the enviroment configuration make sure that the localhost:3000/cable is set as the action cable url.

{% highlight ruby %}
#app/config/enviroments/development.rb
config.action_cable.url = "ws://localhost:3000/cable"
{% endhighlight %}

Finally since we only want logged in users to be the ones who have access to any of the pub/sub data we need to verify them when the connection is made. Let's go ahead and modify our channels connection file. The following snippet is taken from the [rails edge guide]({{http://edgeguides.rubyonrails.org/action_cable_overview.html}}).

{% highlight ruby %}
#app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    protected

      def find_verified_user
        if verified_user = User.find_by(id: cookies.signed[:user_id])
          verified_user
        else
          reject_unauthorized_connection
        end
      end
  end
end
{% endhighlight %}

As you can see this makes a call to cookies.signed to check the user id. We need to make sure we give the user this signed cookie for the verification. Let's do that on login and signup right after we create the session.

{% highlight ruby %}
#app/controllers/users_controller.rb
  def create
    @user = User.new(user_params)
    if @user.save
      session[:user_id] = @user.id
      cookies.signed[:user_id] = session[:user_id]
      @user.rooms << Room.find(1)
      redirect_to rooms_path
    else
      @errors = @user.errors.full_messages
      render "/users/new"
    end
  end
{% endhighlight %}

{% highlight ruby %}
#app/controllers/sessions_controller.rb
  def create
    @user = User.find_by(username: params[:session][:username])
    if @user && @user.authenticate(params[:session][:password])
      session[:user_id] = @user.id
      cookies.signed[:user_id] = session[:user_id]
      redirect_to rooms_path
    else
      @errors = @user.nil? ? ["username not found"] : @user.errors.full_messages
      render "/sessions/new"
    end
  end
{% endhighlight %}

Now let's build out the room channel which will broadcast messages as they come in. rails has a generator for this so we can call. It builds out the javascript channel file and the ruby channel file.

{% highlight powershell %}
rails g channel room
{% endhighlight %}

We only want users to see messages that are sent to the channel that they are currently in so we are going to scope them out to the specific room. Let's build out the client portion first.

{% highlight javascript %}
// app/assets/javascripts/channels/room.js

App.room = App.cable.subscriptions.create({
  channel:'RoomChannel',
   room: window.location.pathname.split("/")[2]
 }, {
  received: function(data) {
    $('#messages').append(data.message);
  }
});
{% endhighlight %}

We send back the name of the room which will link up with the created ruby file. We also send back the param of room which is optional and let's us scope out this channel. You can configure it however you want, I am just going to grab the room id off the end of the url.
as you can see we are going to append the data we receive back through this channel to a messages div. Let's wrap the rendered messages in a div with an id of messages.

{% highlight erb %}
#app/views/rooms/show.html.erb

<div id="messages">
  <%= render @messages %>
</div>
{% endhighlight %}

Now let's make our server side connection. When a user calls the subscription.create method on the front-end, it will call the subscribed method on the specified channel on the backend. Let's wire that up.

{% highlight ruby %}
# app/channels/room_channel.rb
class RoomChannel < ApplicationCable::Channel
  def subscribed
    stream_from "room_#{params[:room]}"
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end
end
{% endhighlight %}

Now we want to broadcast to all of the subscribed users any new created message. We are going to use a job that will perform this after creation. We can use rails generate job for this

{% highlight powershell %}
rails g job MessageBroadcast
{% endhighlight %}

now in our jobs folder lets modify the newly created file.

{% highlight ruby %}
# app/jobs/message_broadcast_job.rb

class MessageBroadcastJob < ApplicationJob
  queue_as :default

  def perform(message)
    ActionCable.server.broadcast "room_#{message.room_id}", message: render_message(message)
  end

  private
    def render_message(message)
      ApplicationController.renderer.render(partial: 'messages/message', locals: { message: message})
    end
end
{% endhighlight %}

Let's put in our message model that any time that a model is created <em>and</em> committed that we want to broadcast it to the room.

{% highlight ruby %}
#app/models/message.rb
  belongs_to :user
  belongs_to :room
  after_create_commit { MessageBroadcastJob.perform_later(self)}
{% endhighlight %}

Now to see if all of this is working we are going to have to ajax our message submit. Let's remove the redirect_to :back in the messages create controller we had previously and build out an ajax call.

{% highlight javascript %}
// app/assets/javascripts/main.js

$('#new_message').on ('submit', function(){
    event.preventDefault();
    // make sure there is content
    if ($('#new_message [name="message[content]"]').val() === ""){
      return false;
    }

    $.ajax({
      url: this.action,
      data: $(this).serialize(),
      method: "post"
    }).done( function(res){
      // reset user input to empty
      $('#new_message [name="message[content]"]').val("");
    });
  });
});
{% endhighlight %}

If everything is working we should have chats being posted in realtime! WHAT! crazy. Test it out, open two browsers, one incognito and login as two different users. You should see it appear on both screens after a submit.

Alright now that we have that working lets get those users displaying in realtime. Now this one is a bit more tricky. Since users can only get data published to them when they are subscribed it's not as simple to show the users that were logged in before you entered the room. Let's add an online field to our memberships table as a way to manage peoples status. create a new migration to add the field online as a boolean to memberships

{% highlight ruby %}
#your migration should look like this

class AddOnlineToMembership < ActiveRecord::Migration[5.0]
  def change
    add_column :memberships, :online, :boolean, default: :false
  end
end
{% endhighlight %}

You should probably drop and remigrate your database at this point.

now create the appearance channel, we are going to scope it out the same way with the room index. Everytime someone subscribes to the room we are going to set the status to true and publish that membership data to the room. Same for when they unsubscribe and we set the status to false. We will first create the appearance channel

{% highlight powershell %}
  rails g channel appearance
{% endhighlight %}

Let's work out the server side code which is going to call two methods that we need to define in the membership model -- is_online and is_offline

{% highlight ruby %}
#app/channels/appearance_channel.rb
class AppearanceChannel < ApplicationCable::Channel
  def subscribed
    member = Membership.where(user_id: current_user.id, room_id: params[:room]).first
    return unless member
    member.is_online
    stream_from "appearance_#{params[:room]}"
  end

  def unsubscribed
    member = Membership.where(user_id: current_user.id, room_id: params[:room]).first
    return unless member
    member.is_offline
  end
end
{% endhighlight %}

{% highlight ruby %}
# app/models/membership.rb add these two methods

  def is_online
    self.update_attributes(online: true)
  end

  def is_offline
    self.update_attributes(online: false)
  end
{% endhighlight %}

Let's make a job similar to how we did for messages after a commit for the membership model. We are going to want to publish data whenever a membership status changes from true to false or vice versa.

{% highlight powershell %}
rails g job AppearanceBroadcast
{% endhighlight %}

{% highlight ruby %}
#app/jobs/appearance_broadcast_job.rb
class AppearanceBroadcastJob < ApplicationJob
  queue_as :default

  def perform(membership)
    ActionCable.server.broadcast "appearance_#{membership.room_id}", render_json(membership)
  end

  private

  def render_json(membership)
    ApplicationController.renderer.render(json: membership)
  end

end
{% endhighlight %}

{% highlight ruby %}
# app/models/membership.rb add this right under the validations
  after_update_commit {AppearanceBroadcastJob.perform_later self}
{% endhighlight %}


Now let's finish up our connection with the javascript channel code. We are going to subscribe to the scoped room, when subscribed we will be waiting for broadcasts from the job we just put together. When we receive that broadcast we are going to use the data to target the li that contains that username and active green dot.

{% highlight javascript %}
// app/assets/javascripts/channels/appearance.js
App.appearance = App.cable.subscriptions.create({
  channel:'AppearanceChannel',
   room: window.location.pathname.split("/")[2]
 }, {
  received: function(data) {
    var membership = JSON.parse(data)
    if (membership.online === true){
      $(userImgIdConstructor(membership)).attr('class', 'active');
    };
    if (membership.online === false){
      $(userImgIdConstructor(membership)).attr('class', 'inactive');
    };
  }
});

var userImgIdConstructor = function(membership){
  return "#" + membership.user_id + "-status";
}
{% endhighlight %}


Now let's switch our rooms/show controller and view to iterate through memberships rather than users so we have access to the online status from the start.

The plan is at first when we load the room show page we are going to display all currently active members by using the membership online status.

{% highlight ruby %}
# app/controllers/rooms_controller.rb

  def show
    @room = Room.find(params[:id])
    redirect_with_flash unless member_of_group

    @messages = @room.messages.order(id: :desc).limit(500).reverse
    @message = Message.new
    @memberships = @room.memberships
  end

{% endhighlight %}

{% highlight ruby %}
#app/views/show.html.erb
#we switch what was previously @users to this

<ul id="users-list">
  <% @memberships.each do |member| %>
      <li>
        <%= member.name %>
        <% if member.online %>
          <%= image_tag "green_dot.png", id: "#{member.user_id}-status", class: "active" %>
        <% else %>
          <%= image_tag "green_dot.png", id: "#{member.user_id}-status", class: "inactive" %>
        <% end %>
      </li>
  <% end %>
</ul>
{% endhighlight %}

Set the css to have the green dot display for the class active and not display for inactive

{% highlight css %}
/* app/assets/stylesheets/main.css */
.active {
  display: ;
}

.inactive {
  display: none;
}
{% endhighlight %}

Now when the pageload initially occurs we see all of the current statuses of users online. That paired with the appearance channel we will see the green dot vanish when users leave the room and appear when they enter.

We have a full fledged chat app at this point, users can talk in realtime and you can view all of the users currently in the room. All that is left to do is style the app. Let's deploy this guy to heroku. Create your new app on heroku and grab the app name.

Now most of this section is taken line for line from [this post]({{https://blog.heroku.com/real_time_rails_implementing_websockets_in_rails_5_with_action_cable}}) by Sophie DeBenedetto. It is a great tutorial and much more informative than this one!

{% highlight powershell %}
heroku create your-app-name
{% endhighlight %}

add redistogo to your app and grab the redistogo url.

{% highlight powershell %}
heroku addons:add redistogo
heroku config --app your-app-name | grep REDISTOGO_URL
{% endhighlight %}

add the redistogo url to your config cable.yml

{% highlight ruby %}
# config/cable.yml
development:
  adapter: async

test:
  adapter: async

production:
  adapter: redis
  url: redis://redistogo:1f287370bc437360cea3f77d7bdc6ede@catfish.redistogo.com:10846/

{% endhighlight %}

Add the following to your config environments production file

{% highlight ruby %}
#
config.web_socket_server_url = "wss://your-app-name.herokuapp.com/cable"
  config.action_cable.allowed_request_origins = ['http://your-app-name.herokuapp.com', /http:\/\/your-app-name.herokuapp.com\/*/]
{% endhighlight %}

that last bit says any address that matches our own with a wildcard for anything to follow. You may want to get more strict on the channels you allow.

specify cable on the client side when you call create consumer

{% highlight javascript %}
// app/assets/javascripts/cable.js

(function() {
  this.App || (this.App = {});

  App.cable = ActionCable.createConsumer("/cable");

}).call(this);
{% endhighlight %}

Now all that's left to do is deploy and pray

{% highlight powershell %}
git push heroku master
heroku run rake db:migrate
heroku run rake db:seed
{% endhighlight %}

Hope that worked out for you!

## Go through all of your before action for validations

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
