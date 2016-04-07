---
layout: post
title: Block News Genius Annotations
date: 2016-04-01
---

Recently, Genius released a product that would allow anyone to annotate any page on the Internet.

There's been a fair bit of media coverage on why such a feature harbors a [potential for harassment](http://thinkprogress.org/culture/2016/03/30/3764816/news-genius-definitely-sounds-like-a-great-idea-cant-imagine-what-could-go-wrong/) or abuse, especially given that content publishers have [no control](http://recode.net/2016/03/28/the-company-formerly-known-as-rap-genius-is-once-again-enmeshed-in-controversy/) over whether their work is annotated.

So, I wrote a gem to block it.

[Genius Blocker](https://rubygems.org/gems/genius-blocker) is a simple piece of Rack Middleware that forces a page redirect should someone attempt to navigate to a site with `genius.it/` prepended to the URL.

I originally set out to accomplish this task by checking the request referer[^1] for a given URL. My plan was to redirect for any URL that listed `genius.it` or similar as the referer. Simple enough, right? I headed over to [Request Bin](http://requestb.in/) to mock out a request from News Genius so that I could inspect its headers.

I was in for an unexpected surprise.

Hitting my Request Bin URL with `genius.it/` prepended onto the URL and returned a response headed indicating a host of `requestb.in` rather than `genius.it`. In other words, it appears that News Genius spoofs the URL so that it appears as though the request is coming from the site meant for annotation, rather than Genius. I'd like to believe that this was simply an oversight, because altering headers like this is poor web etiquette. This post is not intended to be another "The Internet is a shitty place" piece, but it does trouble me that an established company that ostensibly attempts to offer a useful service might deliberately engage in this sort of behavior.

Given this discovery, I knew I needed a different approach. Since it seemed unlikely that I'd be able to deal with the request completely on the server-side, I switched gears to a client-side solution instead.

Enter JavaScript. I updated my tests to check for some script tags that could accomplish this task. This is what I came up with:

{% highlight ruby %}
require 'spec_helper'

describe Genius::Blocker do
  it 'has a version number' do
    expect(Genius::Blocker::VERSION).not_to be nil
  end

  let(:app) do
    Rack::Builder.new do
      use Genius::Blocker
      run lambda { |env| [200, {'Content-Type' => 'text/html'}, [<<-EOT]] }
        <html>
          <head>
          </head>
        </html>
      EOT
    end
  end

  let(:url) { "www.google.com" }
  let(:request) { Rack::MockRequest.new(app) }
  let(:response) { request.get(url) }

  context 'for an html page' do
    it 'adds a script tag to redirect' do
      document = Oga.parse_html(response.body)
      expect(document.css('head > script').size).to eq 1
    end
  end

  context 'for non-html content' do
    let(:app) do
      Rack::Builder.new do
        use Genius::Blocker
        run lambda { |env| [200, {'Content-Type' => 'application/json'}, []] }
      end
    end

    it 'does not redirect' do
      document = Oga.parse_html(response.body)
      expect(document.css('head > script').size).to eq 0
    end
  end
end
{% endhighlight %}

Pretty simple - if a page has the genius URL, a script tag is added to the headers to redirect.

Pop open your web inspector and type `window.location` into the console and check out what gets returned. Go ahead, I'll wait. This tiny bit of JavaScript returns an object with some information about, you guessed it, the current window's location. Though the information in this object isn't quite as informative as what one might get by examining request/response headers, it is more than adequate for our case here. From this, I was able to write a small bit of code to force a redirect based on the presence/absence of a Genius url:

{% highlight javascript %}
<script>
  var annotated = window.location.href.indexOf("genius.it/") !== -1;

  if (annotated) {
    window.location.href = window.location.href.replace("genius.it/", "")
  }
</script>
{% endhighlight %}

First, I created a variable, `var annotated`, which returns a boolean if `genius.it/` is present in the URL. then I force a redirect if `annotated` evaluates to true.

I then wrapped this code in some Rack middleware so that the redirect happens during the page load and added a check to ensure that only html pages are affected. The code, in its entirety, is below. It's worth noting that the JavaScript can be extracted and used on its own for any webpage in the event that your application is not compatible with Rack.

{% highlight ruby %}
require "genius/blocker/version"

module Genius
  class Blocker
    def initialize(app)
      @app = app
    end

    REDIRECT_SCRIPT = <<-EOF
      <script>
        var annotated = window.location.href.indexOf("genius.it/") !== -1;

        if (annotated) {
          window.location.href = window.location.href.replace("genius.it/", "")
        }
      </script>
    EOF

    def call(env)
      status, headers, body = @app.call(env)
      return [status, headers, body] unless html?(headers)

      response = Rack::Response.new([], status, headers)
      body.each do |fragment|
        response.write fragment.gsub(%r{</head>}, "#{REDIRECT_SCRIPT}</head>")
      end

      response.finish
    end

    private

    def html?(headers)
      headers['Content-Type'] =~ /html/
    end
  end
end
{% endhighlight %}

Next up is an effort to make the URL checking more robust by looking for a `hostname` equal to "genius.it". This is a fairly simple switch but is a bit tricker to test given that a script tag will still be present no matter what.

The repository can be found [here](https://github.com/marlabrizel/genius-blocker). I love constructive code reviews; if you have comments, please feel free to leave them in a pull request or GitHub issue!

[^1]: No, this is not a [typo](https://www.quora.com/What-is-the-origin-of-HTTP-header-referer-misspelling).
