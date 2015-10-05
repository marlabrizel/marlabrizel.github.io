---
layout: post
title: Set Up TravisCI to Watch A Rails Repository
date: 2015-10-04
---

<p class="intro"><span class="dropcap">C</span>ontinuous integration is a powerful tool.</p>

[Continuous integration](http://www.thoughtworks.com/continuous-integration), or CI, has many applications. Many professional development shops will use CI to manage production deployments. Because many beginners aren't yet deploying code to production (or are doing so on a very small scale), CI will often get written off as a "later" skill or as something that's too complicated to understand.

However, CI can be incredibly useful, even for the newest of developers. For example, tools such as TravisCI can be configured to watch a repository and automatically run an application's test suite each time a pull request is submitted. Below, I'll walk through the steps to configure TravisCI for a Rails repository.

### I'm new to Rails - why should I care?
Do you work with other people? Probably. No? Have you ever forgotten to run your tests before merging to master? (If you're still staying no at this point, then you're a liar.)

Travis keeps project collaborators on the same page by ensuring that upcoming changes won't cause a regression on master or production. This safety check is helpful whether your repository has less than a thousand lines of code or over a million.

#### Set up a TravisCI account
Head over to [Travis'](https://travis-ci.org/) website and sign up for an account. Do this using your GitHub credentials and you'll automatically be able to connect the repositories you wish to watch. So far, so good.

#### Create a Configuration File

In your project's root directory, create a file called `.travis.yml`. Don't forget the `.` at the beginning of the file name, or the configuration won't take effect. You'll need to check this file into source control, so take care to ensure that it doesn't contain any sensitive information such as API keys.

### Start Configurin'

Our main goal is to ensure that everything is kosher with our repository before we merge our feature branch to master and deploy to production. This means that our migrations make sense, our environment is properly configured, and our test suite is green. We're (hopefully) already doing these things in development, but as we're working (and especially if it's been a while since your last pull request/push to master), our local environment is prone to getting out of sync with what's upstream.

Travis allows us to check our bearings by running everything in a clean, containerized environment. But containerization means that things are running in their own little sandbox, so we need to set things up that are otherwise already running on our dev machines.

##### Write a basic build script

Let's assume that the only thing we want to do is just run the tests. Easy enough, right? Yes, but first we've got to go through the same steps we would if we were running `rails new` on our local machine.

{% highlight ruby %}
language: ruby
rvm:
  - 2.2.2

script:
  - bundle install
  - bundle exec rake db:setup
  - bundle exec rake
{% endhighlight %}

Here we're specifying a language (Ruby) and a version (2.2.2) for our build. These should match your dev environment. Note that the one thing that *doesn't* need to match is your choice of version management. For example, I use `rbenv` to manage ruby versions on my local machine, but the Travis VMs use `rvm` so I specify this version manager in my configuration.

After adding language and version, we giving Travis a script to execute. `bundle install` installs all of the gems specified in the Gemfile, and `bundle exec rake db:setup` creates and migrates the database as well as runs the seed file should it exist. Finally `bundle exec rake` runs your test suite. I recommend using the rake command as it's test framework-agnostic, meaning you won't have to update your Travis scripts should you some day choose to move from Test::Unit to RSpec or vice-versa.

##### Trigger a build

Commit your changes to this file and push to trigger your first build. If all goes well, each phase of the build script will exit with `status: 0`. Congrats! You now have Travis running.

##### Configure API keys

If all you have is a vanilla Rails project without any bells and whistles, you should be set. But things will blow up as soon as you start layering on additional dependencies. What happens, say, if your application uses a 3rd-party API? (*You don't have your keys checked into version control, do you?*) Since your credentials are stored locally somewhere, you'll need to somehow pass them to Travis so that any API calls made in the tests can run.

The good news is that, if you're using VCR and have checked the cassettes into version control, Travis will read these cassettes when running the tests. Because the tests are running from the cassettes, no external calls are being made, so you can get away with stubbing out an API key. Travis will read your keys from a set of global environment variables; therefore, the key names will still need to match what you have defined elsewhere.

{% highlight ruby %}
language: ruby
rvm:
  - 2.2.2

env:
  global:
    - MY_API_KEY=abc123
    - MY_CLIENT_ID=1234
    - MY_CLIENT_SECRET=def456

script:
  - bundle install
  - bundle exec rake db:setup
  - bundle exec rake
{% endhighlight %}

Here I've added the global environment variables needed to access my API. Note that the names match how I've defined the variables across my application, but I've stubbed out the actual values. If you have done this properly, your build scripts will exit with a 0 status; else, the `bundle exec rake` portion of the script will likely blow up with an indication that VCR does not know how to handle your requests.

##### API keys: a side note
If you're using VCR to run tests and haven't configured it to sanitize your keys in your cassettes, your efforts to mask them in your Travis configuration are effectively a waste. You'll want to add the following to your VCR configuration:

{% highlight ruby %}
VCR.configure do |config|
  ... # your VCR config here
  config.filter_sensitive_data('<MY_API_KEY>') { ENV['MY_API_KEY'] }
  config.filter_sensitive_data('<MY_CLIENT_ID') { ENV['MY_CLIENT_ID'] }
  config.filter_sensitive_data('<MY_CLIENT_SECRET>') { ENV['MY_CLIENT_SECRET'] }
end
{% endhighlight %}

This addition will replace the value associated with the environment variable passed in each block with the string specified between brackets. If you don't do this, your API keys and other sensitive data will be recorded in plain text in your cassettes (oops). You'll want a separate line for each piece of data you wish to sanitize.

##### Configure an additional database

If your application uses an additional database, you'll need to tell Travis about that too. Here we'll add Redis to the mix, but the [Travis docs](http://docs.travis-ci.com/user/database-setup/) provide guidance on many different options.


{% highlight ruby %}
language: ruby
rvm:
  - 2.2.2

env:
  global:
    - MY_API_KEY=abc123
    - MY_CLIENT_ID=1234
    - MY_CLIENT_SECRET=def456

script:
  - bundle install
  - bundle exec rake db:setup
  - bundle exec rake

services:
  - redis-server
{% endhighlight %}

Additional databases are recognized as services, so here we've added Redis. Pretty easy.

#### Is there more?

Of course. This post covers the basics but as you continue to customize your environment, you'll need to update Travis. Fortunately, the [docs](http://docs.travis-ci.com/) are extremely easy to follow and will get you to where you need to be.
