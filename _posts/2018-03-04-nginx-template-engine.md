---
layout:     post
title:      "Hosting Template Engine Server with Nginx and Openresty's Lua Plugin"
date:       2018-03-04 17:30:20 +0530
comments:   true
---

We wanted to have a server which can serve dynamically generated HTMLs from templates based on request. One can write this as a webapp in any web-framework and deploy on http server; but why not program the server itself!

### The problem
We want server to return basic html from template. Something like whenever the `http://localhost:8090/myapp?param1=hello&param2=John` is called it should return `text/html` response which looks like this:
```html
<html>
<body>
    <p>Nice to see you John. Platform greets you "hello".</p>
</body>
</html>
```

The name and greeting is compiled using a template from param. so the template looks like this:
```html
<html>
<body>
    <p>Nice to see you {{param1}}. Platform greets you "{{param2}}".</p>
</body>
</html>
```

### Nginx
Anyone working in web development would have heard about [Nginx](https://www.nginx.com/) - most likely as the highly available server for setting up reverse proxy for load balancing. It can be used for many other services as well. Until recently I was using Nginx as just the server container with a simple config file which one can use to _configure_. After I met an Nginx module by Openresty for [lua-nginx](https://github.com/openresty/lua-nginx-module) - I can use Nginx to _program_.

> OpenResty turns the NGINX server into a powerful web app server, in which developers can use the Lua programming language to script various existing nginx C modules and Lua modules and construct extremely high-performance web applications that are capable to handle 10K ~ 1000K+ connections in a single box.

### The solution
```bash
brew install openresty/brew/openresty
brew install pcre openssl curl

mkdir myapp
cd myapp

mkdir logs/ conf/ templates/ lua/
```

#### templates/greet.html
```html
<html>
<body>
     <p>Nice to see you {{param1}}. Platform greets you "{{param2}}".</p>
</body>
</html>
```

#### installing lua's templating library
One greatest benefit of using lua-nginx module would be able to use [numerous existing libraries](https://github.com/bungle/awesome-resty) for programing our app.
We would use [lua-resty-template](https://github.com/bungle/lua-resty-template) to compile the html templates. Installing library using OpenResty package manager (opm):
```bash
opm get bungle/lua-resty-template
```

#### lua/myapp.lua
```lua
local template = require("resty.template")
local template_string = ngx.location.capture("/templates/greet.html")
template.render(template_string.body, {
    param1 = ngx.req.get_uri_args()["param1"],
    param2 = ngx.req.get_uri_args()["param2"]
})
```

Bunch of things are used here. `ngx` is global variable we can use in our lua files to communicate with openresty's nginx server.

#### conf/nginx.conf
```nginx
worker_processes  1;
error_log logs/error.log;
events {
    worker_connections 1024;
}
http {
    root ./;
    server {
        listen 8090;

	location /myapp {
		default_type text/html;
		content_by_lua_file ./lua/myapp.lua;
	}
}
```

#### run
```bash
./usr/local/openresty/nginx/sbin/nginx -p ~/myapp -c conf/nginx.conf
```

That should run the myapp server on 8090 port. And `curl http://localhost:3000/myapp?param1=hello&param2=John` should give the compiled HTML we wanted from our template.

### TL;DR
Plain Nginx is fast and easy to configure. OpenResty gives enhances support on Nginx; in a way it is Nginx++. This gives power to program Nginx with lua scripting - which is also fast and easy. One can make Nginx compile templates, do access validation, auth verification all in Nginx itself, even when configuring it as reverse proxy for a webapp.

### References:
1. https://yos.io/2016/01/28/building-an-api-gateway-with-lua-and-nginx/
1. https://www.staticshin.com/top-tens/things-about-openresty.html
1. https://www.staticshin.com/programming/definitely-an-open-resty-guide/
1. https://openresty.org/en/installation.html
1. https://www.digitalocean.com/community/tutorials/how-to-use-the-openresty-web-framework-for-nginx-on-ubuntu-16-04
1. https://github.com/openresty/lua-nginx-module
1. https://github.com/bungle/lua-resty-template
1. https://stackoverflow.com/questions/19853930/getting-error-while-using-lua-with-nginx
1. https://github.com/bungle/awesome-resty
