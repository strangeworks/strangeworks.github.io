---
layout: post
title: "Bots on Rails. Part one: looking for an idea and setting things up."
tags: [ruby, rails]
---
{% include image.html path="posts/bots-header-1.png" path-detail="posts/bots-header-1.png" alt="Bots header" %}

Messenger bots still one of the most hot discussed 
topic in tech community. Buzzwords such as «conversational commerce» can not be unseen.

### So what’s messenger bot really is? 
Conceptually messenger bots are simple webhook
handling services, that can be done in every 
technology you prefer. I personally prefer rails, 
and in this introductory tutorial I would like to share some concepts 
how fun building Facebook messenger bot can be on Rails.

### Everything starts from an idea.
{% include image.html path="posts/idea.png" path-detail="posts/idea.png" alt="Idea of bot" %}

Currently across the web there are bunch of naive bot examples that shows 
weather forecasts, food delivery, or simple echo services. 
I wouldn’t say that weather forecast bot examples is boring 
but we would like to build something where we can both play with 
as many interactions as possible and touch some fancy stuff 
from Facebook messenger API. So what are we going to build? 
We want to build something less boring and doable in the same time. 
And the first thing that came to my mind was concept of life log 
where you prompted with a simple question and just send small 
sentence about your day. Sounds pretty simple, huh? So let’s first 
describe our user flow:

* When I write message to bot I would like start conversation and get some introductory message.

* After that I would like to see what bot can do.

* Then after success on-boarding I would like to setup it first

* During this setup phase bot will prompt me when I would like to be prompted to give life log sentence of this day.

* Then I would like to be asked about my mood, and I can pick it from emoji smileys list.

* After that bot will respond with some clever answer and store this message for future reference.

This flow looks very simple and straightforward, 
how can we make it more fancier? Let’s add additional requirement, 
I would like to ask bot in natural language: “What I did two days ago?”, 
and bot will send me clickable button that will open web view with my activity page.

### Preparing our toolbox
{% include image.html path="posts/toolbox.png" path-detail="posts/toolbox.png" alt="Bot toolbox" %}
Fortunately, thanks to great ruby and rails community we 
will not start building our own Facebook messenger API payload 
parsers and we will use very great library 
[hyperoslo/facebook-messenger](https://github.com/hyperoslo/facebook-messenger), 
which gives the most important 
toolset out of the box. Returning to our requirements 
we will also need to find out how to parse natural language 
and provide proper responses from bot to end user, and fortunately 
for this we can leverage some awesomeness of wit.ai, that gives pretty 
great natural language api absolutely for free.

### Building our dummy bot
We have all the needed toolset, so the main question is 
how to get started? What do we need first, 
let’s think about our checklist:

* Facebook requires to have a separate Facebook page, and facebook app that can be created in facebook developer center

* To start processing messages we will need service that will handle webhooks sent from facebook, for this purpose will write our rails app.

* Our bot requires some persistence layer to ‘remember’ all settings and data the we will provide from conversation flow, so we will need some database, in our case we can use heroku.

#### Setting everything up on Facebook side.
{% include image.html path="posts/facebook-page-setup.png" path-detail="posts/facebook-page-setup.png" alt="Idea of bot" %}
* First you need to create your own Facebook page which will represent your bot. You can do it by visiting this link

* In order to start working with webhook processing we need to create an Facebook app that can be done by following developers.facebook.com.

{% include image.html path="posts/facebook-add-product.png" path-detail="posts/facebook-add-product.png" alt="Add facebook product" %}

* After you have created your app you will need to add messenger platform support, this can be done by clicking add product menu item at the sidebar. And now you will have almost everything you need from facebook side, we will return to developer center a bit later and now we will need finally to dive into our rails app.

#### Building simple rails app.
I will not describe everything what you need to do for setting up rails app, 
I expect that you have already familiar with rails and can create new rails app by yourself. 
If you don’t want to do this you can also clone [this repo](https://github.com/strangeworks/lifelog-bot) where all the starter files are already provided. After you setup rails and all required dependencies you will need to make sure that all required dependencies works properly.

Make sure that you have added 
```gem 'facebook-messenger'``` 
to your Gemfile As we are using custom directory 
for bot files we need to make sure that rails “knows” about it.

```ruby
# config/initializers/messenger_bot.rb

 unless Rails.env.production?
   bot_files = Dir[Rails.root.join('app', 'messenger_bot', '**', '*.rb')]
   bot_reloader = ActiveSupport::FileUpdateChecker.new(bot_files) do
     bot_files.each{ |file| require_dependency file }
   end
 
   ActionDispatch::Callbacks.to_prepare do
     bot_reloader.execute_if_updated
   end
 
   bot_files.each { |file| require_dependency file }
 end
 
# config/application.rb
require_relative 'boot'

require "rails"
# Pick the frameworks you want:
require "active_model/railtie"
require "active_job/railtie"
require "active_record/railtie"
require "action_controller/railtie"
require "action_mailer/railtie"
require "action_view/railtie"
require "action_cable/engine"
require "sprockets/railtie"

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)
Dotenv::Railtie.load

module LifelogBot
  class Application < Rails::Application
    # Settings in config/environments/* take precedence over those specified here.
    # Application configuration should go into files in config/initializers
    # -- all .rb files in that directory are automatically loaded.
    config.paths.add File.join('app', 'bot'), glob: File.join('**', '*.rb')
    config.autoload_paths += Dir[Rails.root.join('app', 'bot', '*')]
  end
end


# config/routes.rb
Rails.application.routes.draw do
  mount Facebook::Messenger::Server, at: 'webhooks/messenger'
end

```


In order to successfully communicate with facebook api 
from you dev environment, you need to share access to your 
local machine somehow, and for this purposes I would suggest 
to use pretty great tool ngrok. Run your rails server using 
```bundle exec rails server``` and 
then open new shell and run ```ngrok http 3000``` 
where 3000 it can be any port where your rails app 
is currently bound. Quick note: if you are starting your app 
at first time don’t forget to run ```rails db:create```

The next thing you will need are tokens from 
facebook developer console, you will need to get: 
page access token, verify token for incoming webhooks 
and app token, all these tokens you can get from developer 
console.

As I have provided .env support to 
the starter repo you can add .env file and include the 
following keys there

```bash
ACCESS_TOKEN=YOURTOKEN
APP_SECRET=YOURSECRET
VERIFY_TOKEN=YOURVERIFYTOKEN
```

{% include image.html path="posts/webhook-explained.png" path-detail="posts/webhook-explained.png" alt="Idea of bot" %}
After you have successfully obtained all the required tokens you should add your 
url provided by ngrok to the facebook webhook settings.

After that, let’s write some basic code to test if everything works.

```ruby

require 'facebook/messenger'

include Facebook::Messenger

Bot.on :message do |message|
  message.id          # => 'mid.1457764197618:41d102a3e1ae206a38'
  message.sender      # => { 'id' => '1008372609250235' }
  message.seq         # => 73
  message.sent_at     # => 2016-04-22 21:30:36 +0200
  message.text        # => 'Hello, bot!'
  message.attachments # => [ { 'type' => 'image', 'payload' => { 'url' => 'https://www.example.com/1.jpg' } } ]

  message.reply(text: 'Hello, human!')
end
```
Then we can restart our rails server, 
find our facebook page name in messenger and simply write a 
message in our case let it be ***hello bot*** . According to the code 
example we should respond with simple: ***hello human***

{% include image.html path="posts/hapiness.png" path-detail="posts/hapiness.png" alt="hapiness" %}

Yay! We have basic skeleton for our bot!

#### What’s next?
In this part we have successfully defined scope of our small project, provided basic bot structure, prepared all required dependencies so that we can move forward. In the next parts we will deep dive into implementation of setup phase we will touch different messenger specific ui elements and even implement basic natural language processing support. See you in the next part!
