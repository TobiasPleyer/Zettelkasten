---
date:  2023-11-16
tags:
---

# NGINX with single page apps

taken from https://sdickinson.com/nginx-config-for-single-page-applications/

see also [[202311162116-nginx-front-controller-pattern]]

NGINX config for Single Page Applications

Whether you're using create-react-app, Next.js or even something non React
based (like Angular), you'll come across the case where you write a Single Page
Application (SPA) and need to support linking into a particular page.

Default webserver config doesn't really support this very well.  eg.

If you have a page like https://example.com/ and want to link directly to a
users page like https://example.com/users/ it typically will give you a 404
error as there is no index.html in a users directory on the server. The
cheapout option is to change your links from paths to anchors (eg. https://
example.com#users) but this is ugly, plus prevents you using them to link to a
section on the page (ie. their intended usage).

So here's the config I came up with for my photography site.

http {
    # what times to include
    include       /etc/nginx/mime.types;
    # what is the default one
    default_type  application/octet-stream;

    # Sets the path, format, and configuration for a buffered log write
    log_format compression '$remote_addr - $remote_user [$time_local] '
        '"$request" $status $upstream_addr '
        '"$http_referer" "$http_user_agent"';

    server {
        # listen on port 80
        listen 80;
        # save logs here
        access_log /var/log/nginx/access.log compression;

        gzip on;
        gzip_types text/html application/javascript application/json text/css;

        # where the root here
        root /usr/share/nginx/html;
        # what file to server as index
        index index.html;

        location / {
            # First attempt to serve request as file, then
            # as directory, then fall back to redirecting to index.html
            try_files $uri $uri/ $uri.html /index.html;
        }

        location ~* \.(?:css|js|jpg|svg)$ {
            expires 30d;
            add_header Cache-Control "public";
        }

        location ~* \.(?:json)$ {
            expires 1d;
            add_header Cache-Control "public";
        }
    }
}

The interesting parts of this are the location sections.

location / is what it makes all work, the try_files directive basically says to
attempt to serve it directly, then fall back with a / (ie. to serve an
index.html from a directory), then try a html file with the same path name,
finally just serve up the normal index.html. This last step is what makes the
SPA work, typically the main page will do your routing and looks at the URL to
do it, beyond that it doesn't care where it's served from.

The second and third location sections are overrides to set up caching for
images, javascript, and css files, along with json files separately. In the
case of my site, images, javascript and css won't change (and if the js or css
do, they get new filenames), so we cache for 30 days, however, the json files
will update whenever I add a new image to the site, rebuild, and push. But I do
want some caching, so 1 day seems reasonable.
