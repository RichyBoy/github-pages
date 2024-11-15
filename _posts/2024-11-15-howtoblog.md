---
title: "How to setup a blog like this one.."
date: 2024-11-15
layout: default
tags: jekyll blog github vscode
---

# An obligatory blog how-to post

I was inspired by a blog article created by a friend, his blog can be found [here](https://wmcdonald404.co.uk/). Effectively it's in three stages: create a repo on github in the github.io project, go through the tutorial which is generated for you when creating the repo (and it is super easy) and if you want to see changes rendered locally you can use Visual Code and Jekyll running in a Docker container (also pretty easy to setup).

## Dive straight in

Dive straight in with the quickstart detailed here: [Github.io quickstart](https://docs.github.com/en/pages/quickstart).

## Writing some content

Markdown is straight-forward particularly as nearly everyone has formatted a wiki article before now.. try this guide [here](https://www.markdownguide.org/basic-syntax/). I personally makes changes into a branch and then pull-request the branch onto main when I'm ready - no need to do that but the rigour is good practice.

## Displaying content locally

If you are like me then you'll enjoy rendering your work locally before you push. The pre-requisite is a docker deamon on your windows machine, and Visual Code. My friend also has a blog page on this [here](https://wmcdonald404.co.uk/2024/10/17/running-jekyll-in-a-devcontainer.html). If you follow that guide the only missing bit of info is that the config file you need is called devcontainer.json and it lives in the .devcontainer directory that you create. This whole process took me around 5 minutes - dead simple.
