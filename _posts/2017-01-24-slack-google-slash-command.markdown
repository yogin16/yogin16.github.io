---
layout:     post
title:      "Creating Slack slash command for Google search using Node and Heroku"
date:       2017-01-24 12:34:58 +0530
comments:   true
---
I love Slack. I feel it is a great product. [The company strives for best product](https://slack.engineering/data-wrangling-at-slack-f2e0ff633b69#.qxhj9rsjh). And I love Google, as everyone does. So I decided to add google search within Slack.

This was first interaction playing with Node and Heroku. and I became fan of both. Here is how I added web search slash command in two days for Slack.

### Setting up the server
The first thing needed was the server. Server which would listen to the webhook requests from Slack and perform google search. I explored Node's express.js framework to create [server](https://github.com/yogin16/slack-google) with Node's Google kit.
Once the server was ready, deploying and hosting it on Heroku was easier than expected. just needed to connect Heroku with Github and some commands here and there and the server was ready. Heroku has great UI as well as CLI support along with great documents.

![server hosted]({{ site.url }}_assets/images/heroku_dashboard.png)

It is hosted on (https://google-slack.herokuapp.com/)[https://google-slack.herokuapp.com/]

### Add the slash command
As Slack allows us to add custom integrations and leverage power of adding own slash commands very easily, the next step was not very time consuming.
Just have to add the slash command configuration - it asks for **command name**, **the server url**, **description**

![slash command added]({{ site.url }}_assets/images/slack_slash_command.png)

### It works!
![We can search google directly from Slack.]({{ site.url }}_assets/images/slack_google_search.png)