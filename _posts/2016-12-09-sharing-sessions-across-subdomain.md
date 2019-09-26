---
layout: post
title: "Sharing sessions across subdomain for Rails"
date: 2016-12-09 15:06:57 +0800
comments: true
categories: [Web]
tag: [Rails, Session]
---

To share the sessions across subdomains for a Rails a app, you may set the configuration like this as the documentation suggest:

``` ruby
App.config.session_store ... , :domain => :all
```

Unfortually, it does’t work unless you’re visit by localhost.

The `:all` defaults to a TLD length of 1, which means if you’re testing with Pow (myapp.dev) it does’t work because the TLD is 2:

``` ruby
App.config.session_store ... , :domain => :all, :tld_length => 2
```

<br />

---

_**[References]**_

* http://excid3.com/blog/sharing-a-devise-user-session-across-subdomains-with-rails-3/
* http://stackoverflow.com/questions/10402777/share-session-cookies-between-subdomains-in-rails/15009883#15009883
