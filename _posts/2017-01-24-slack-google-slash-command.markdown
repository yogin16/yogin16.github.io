---
layout:     post
title:      "Creating Slack slash command for Google search using Node and Heroku"
date:       2017-01-24 12:34:58 +0530
comments:   true
---
I love Slack. I feel it is a great product. [The company strives for best product](https://slack.engineering/data-wrangling-at-slack-f2e0ff633b69#.qxhj9rsjh). And I love Google, as everyone does. So I decided to add google search within Slack.

This was first interaction playing with Node and Heroku. and I became fan of both. Here is how I added web search slash command in two days for Slack.

### Setting up the server
The first thing needed was the server. Server which would listen to the webhook requests from Slack and perform google search, give the search results back. NPM has [Google](https://www.npmjs.com/package/google) kit. So I was hoping to use that and deploy the Node server. I explored Node's [express.js](http://expressjs.com/) framework to create [server](https://github.com/yogin16/slack-google).

1. Setting up [express.js](http://expressjs.com/en/starter/installing.html) repo.

```bash
$ mkdir slack-google
$ cd slack-google

$ npm init
$ npm install express --save
```

2. Get additional libs that we would require

```bash
$ npm install google
$ npm install lodash
```

3. Add app.js and define routes (Equivalent to adding a servlet as I am coming from Java world)

```javascript
var express = require('express');
var google = require('google');
var _ = require('lodash');
var app = express();

app.get('/', function (req, res) {
  execute(req, res);
})

app.post('/', function (req, res) {
  execute(req, res);
})

app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
})
...
```

That was all. Added the business logic in `execute` and the server is ready. we can do `node app.js` on local directory and it would be listening on the port.

### Hosting the server
Once the server was ready, deploying and hosting it on Heroku was easier than expected. Just needed to connect Heroku with Github and some commands here and there and the server was ready. Heroku has great UI as well as CLI support along with great [document](https://devcenter.heroku.com/articles/git).

One common problem I faced while setting up Heroku - it would not serve the requests and give 500 error. That was because the mistake missing key steps in the process.

- Heroku need to know the start up command for the server process. Need to add `Procfile`. It contains the instruction to run commands on deployment. our content is: `web: node app.js` as we have node server.
- The app is not going to be hosted on 3000 port. the common http port is 80 and https is 443. Changing `app.listen` to as:

```javascript
var PORT = process.env.PORT || 3000;
app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
})
```

Server hosted.
![server hosted]({{ site.url }}assets/images/heroku_dashboard.png)

It is hosted on [https://google-slack.herokuapp.com/](https://google-slack.herokuapp.com/) - our server url.

### Add the slash command
As Slack allows us to add custom integrations and leverage power of adding own slash commands very easily, the next step was not very time consuming.
Just have to add the slash command (has to be done by admin) configuration in `team > apps & integration > build` - it asks for **command name**, **the server url**, **description**

Slash command added.
![slash command added]({{ site.url }}assets/images/slack_slash_command.png)

### It works!
We can do google search directly from Slack.
![We can search google directly from Slack.]({{ site.url }}assets/images/slack_google_search.png)