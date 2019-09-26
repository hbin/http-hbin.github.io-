---
layout: post
title: "Prevent Nginx From Processing Requests with Undefined Server Name"
date: 2016-07-26 00:12:50 +0800
comments: true
categories: [Web]
tag: [Nginx]
---
It makes me confused for a while, so I decide to write it down.

The first server block in the nginx config is the default for all
requests that hit the server for which there is no specific server
block.

A proper Nginx configs have a specific server block for defaults

``` nginx
http {
    [...]

    server {
        listen 80 default_server;
        server_name _;
        return 444; # HTTP response that simply close the connection and return nothing
    }

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

<br />

---

_**[References]**_

- [http://nginx.org/en/docs/http/request_processing.html](http://nginx.org/en/docs/http/request_processing.html)
- [http://stackoverflow.com/questions/9824328/why-is-nginx-responding-to-any-domain-name](http://stackoverflow.com/questions/9824328/why-is-nginx-responding-to-any-domain-name)
- [http://security.stackexchange.com/questions/55790/nginx-how-to-prevent-processing-requests-with-undefined-server-names-with-http](http://security.stackexchange.com/questions/55790/nginx-how-to-prevent-processing-requests-with-undefined-server-names-with-http)
