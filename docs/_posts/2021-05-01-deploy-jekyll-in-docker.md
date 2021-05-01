---
layout: post
title:  "Deploy Jekyll in Docker"
categories: jekyll 
tags: jekyll docker github-pages windows
---

In a powershell terminal run all of these commands:

{% highlight powershell %}
cd [repository directory]
docker run --rm --volume=$(pwd):/srv/jekyll -p 4000:4000 -it jekyll/jekyll:latest bash
chown jekyll:jekyll -R /usr/gem
jekyll new docs
cd docs
bundle update
jekyll build
jekyll serve
{% endhighlight %}