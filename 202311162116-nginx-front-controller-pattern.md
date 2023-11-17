---
date:  Thursday, November 16, 2023
tags:
---

# NGINX Front Controller Pattern

Normally when the URI of a website changes this results in a request to the backend, which returns the .html file
"belonging" to the request route.

But what if your website actually is a single page app (SPA)? In such a case all routing is handled in the frontend and
there is only one source, the complete app.

This is what NGINX refers to as the [front controller pattern](https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/#front-controller-pattern-web-apps).

```
try_files $uri $uri/ $uri.html /index.html;
```

Basically you tell NGINX if you can find the specific target of the URI (what is the case for a SPA) then try the
index file, which is the app. The app then handles the route target and serves the right page.

[[202311162112-NGINX with single page apps]] details this further.
