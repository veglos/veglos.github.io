---
layout: post
title:  "Deploy Jekyll in Docker"
categories: howto 
tags: jekyll docker github-pages
---

[GitHub Pages](https://pages.github.com/) is a fantastic hosting service for blogging, specially for software developers. GitHub Pages runs on [Jekyll](https://jekyllrb.com/) and I did not want to install Ruby and other dependencies on my Windows system. Consequently, docker was the answer.

On a PowerShell terminal

```powershell 
cd [path to the project directory]
docker run --rm --volume=$(pwd):/srv/jekyll -p 4000:4000 -it jekyll/jekyll:latest bash
```

On the container's bash terminal we must type the following. It its important to **chown** because otherwise we will get permission errors.

```bash
chown jekyll:jekyll .
chown jekyll:jekyll -R /usr/gem
jekyll new .
bundle update
jekyll build
jekyll serve
```
The project now is hosted in **http://localhost:4000/**

It is convenient to create a permanent container to serve our project.The option **---force_polling** enable jekyll to watch for changes and automatically deploys them.

```powershell
docker run --name myblog --volume=$(pwd):/srv/jekyll -p 4000:4000 -it jekyll/jekyll:latest jekyll serve --force_polling
```

Next time we just start up the container.