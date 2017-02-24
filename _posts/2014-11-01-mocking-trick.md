---
layout: post
title: "Minitest mocking trick"
description: "TIL mocking trick with ruby minitest framework"
tags: [ruby]
---

Watching pbp video by @tenderlove, I was impressed how it is possible 
to implement some mocking stuff without any dependecies. 
Take a look.

{% highlight ruby %}
module SomeMod
  class SomeModClientTest < MiniTest::Unit::TestCase
    def test_awesomeness
      tc = self
      klass = Class.new(ModClient) do
        define_method(:long_task) do |arg|
          tc.assert_equal 'ponies', arg
        end
      end
      data = klass.method_to_test 'ponies'
      asert_equal data.kitten, ':3'
    end
  end
end
{% endhighlight %}