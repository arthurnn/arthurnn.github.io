---
layout: article
title:  "Fixtures and Factories. Why not both?"
date:   2016-09-10 00:00:00 -0400
categories: ruby
---

Have you ever wondered why some people are so strong about liking fixtures as opposed to factories when testing their Rails apps?
The main advantages/disadvantages points I see about fixtures are:

- 👍  They are fast.
- 👍  You have a full representation of your models, and can just use them without much setup.
- 👎  Every time I change a column I need to update a bunch of fixtures.
- 👎  I have a lot of similar fixtures that just change one or two attributes among them.

Those are all valid points, and I agree with them. I am a fan of fixtures and have used them for a long time. However at GitHub, we don't use YAML files for fixtures, we prefer using Factories to create some test data before our tests run.
However, Factories have some drawbacks too, and the main one, in my opinion, has to do with performance. 

{% highlight ruby %}
class UserTest < ActiveSupport::TestCase
  setup do
    @user = FactoryGirl.create(:user)
  end

  test "username" do
    assert_equal "arthurnn", @user.username
  end
end
{% endhighlight %}

This is how some people utilize factories. You would create your model  in a setup block, and use that in your test. The issue about that, though, is that setup block will run before each test, that means that every new test you add, you are increasing the total time that test file will run.


How about if you could use Factories, but load them in the same way fixtures get loaded? 

{% highlight ruby %}
class UserTest < ActiveSupport::TestCase
  fixtures do
    @user = FactoryGirl.create(:user)
  end

  test "username" do
    assert_equal "arthurnn", @user.username
  end

  test "username2" do
    @user.update!(username: 'arthurnn2')
    assert_equal "arthurnn2", @user.username
  end
end
{% endhighlight %}


The idea in there is to put the model load code inside a fixtures block, have that block been executed only once. And for every test, before we open a transaction to run the test, we would reload the instance variables that were defined in the fixtures block.
With that, you can now use factories, but load them as they were fixtures, so they will be fast (only load them once).

The implementation to make that possible is one module that needs to be included in your test, and it will override some default fixtures functionality. I tested that code on Rails 3.2 and Rails 5.0, so it should work for the versions in between them too. For now, I am leaving that as a simple file/module, but if I see people are interested in using it, I could extract it to a gem and add more functionality, for instance, allow to load Factories and fixtures file at the same time.

The entire code is in this [gist](https://gist.github.com/arthurnn/ad5aa2b6811dfddaf94fa35aed84a73c).
