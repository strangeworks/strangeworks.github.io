---
layout: post
title: "Rspec shared examples tip"
description: "Quick rspec shared examples tip"
tags: [ruby]
---

Sometimes I would like to use one shared examples in different controller actions, 
but how to share failed state where we are checking either user authenticated or not? 
Today I found out a pretty neat way of test suite organization using rspec shared examples. 
Let’s assume that we have a controller where non-authenticated user should be redirected to sign_in_path, 
and devise gem, for example, provides for us pretty good implementation of action before_action callback. 
How can I provide shared example for this case? Looking for some examples I found out a great idea of passing block 
to shared example group with distinct implementation of setup stage for any special context. 
In my example it would look like this:


{% highlight ruby %}
RSpec.shared_example "fails without authentication" do
  context "unauthenticated" do
    it "should redirect to new session path" do
      make_request

      expect(response).to redirect_to(new_user_session_path)
    end
  end
end
{% endhighlight %}

and for our rspec controller we do something like this:

{% highlight ruby %}
RSpec.describe ProjectsController, :type => :controller do
  describe "#index" do
    it_behaves_like "fails without authentication" do
      def make_request
        get :index
      end
    end
  end
end
{% endhighlight %}

As you see we defined make_request method in context of shared example.