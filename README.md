# nextcloudwithcollaboraindocker
Installation of Collabora in Docker for Nextcloud

# Collabora
The internal Collabora server is not very powerful and quickly reaches its limits. On the other hand, it runs much more stable than OnlyOffice. Nevertheless, it was decided to install it outside the cloud by means of a ReverseProxy. This is the same as the [installation of OnlyOffice](https://github.com/Fiodin/nextcloudwithooindocker).

## Docker Image - Collabora
Basically, a Docker image is simply installed as before. Just make sure that the ports are open and not the same as for Only Office:
> docker run -i -t -d -p 127.0.0.1:port:port -e "aliasgroup1=https://sub.domain.endung:443,https://sub.domain.endung" -e username="unique_name" -e password="secure_password" --restart always --name="unique_name" collabora/code

**Important:**
- *name* must be written in "".
- A free local port must be used. This port must also be released in the firewall (e.g. `ufw`) and the router.
- With the domain, care must be taken with the 2-fold \\ and set BEFORE the dots in each case.
- The `password` should not include any quotations as `"` or `'`. Otherwise, Docker interprets this as the end of the string.

If two clouds are to run with the same image, simply extend the domains:
> -e domain="sub\.domain\\.ending|sub\.2tedomain\\.ending"

## Setting up the reverse proxy with NGINX
In order for the Collabora server to be accessible, the reverse proxy must be set up in NGINX.

The setup consists of three steps:
1. Conf for reachability with port 80.
2. create LE certificate with Certbot
3. rewrite Conf for the reverse proxy

For normal accessibility with port 80, simply write a normal Conf for NGINX. You can use anything that is currently available:
>
    server {
        listen 80;
        listen [::]:80;

        server_name sub.domain.extension;

        access_log /var/log/nginx/codeserver-access_log;
        error_log /var/log/nginx/codeserver-error_log crit;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;
    }

After that the certbot can run. If `--nginx` is given as an option, the corresponding changes will be taken over.

Finally, adjust the conf to the reverse proxy. You have to add a `location` section with the proxy pass information. This is a little more complicated if the actual domain can be reached via HTTPS. I have listed the full conf here to show what changes Certbot has made:
>
    server {
        server_name sub.domain.extension;

        access_log /var/log/nginx/codeserver-access_log;
        error_log /var/log/nginx/codeserver-error_log crit;

        listen [::]:443 ssl http2; # managed by Certbot
        listen 443 ssl http2; # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/way.to/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/way.to/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

        # static files
        location ^~ /loleaflet {
                proxy_pass https://localhost:port;
                proxy_set_header host $http_host;
        }

        # WOPI discovery URL
        location ^~ /hosting/discovery {
                proxy_pass https://localhost:port;
                proxy_set_header host $http_host;
        }

        # Capabilities
        location ^~ /hosting/capabilities {
                proxy_pass https://localhost:port;
                proxy_set_header host $http_host;
        }

        # main websocket
        location ~ ^/lool/(.*)/ws$ {
                proxy_pass https://localhost:port;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "Upgrade";
                proxy_set_header Host $http_host;
                proxy_read_timeout 36000s;
        }

        # download, presentation and image upload
        location ~ ^/lool {
                proxy_pass https://localhost:port;
                proxy_set_header host $http_host;
        }

        # admin console websocket
        location ^~ /lool/adminws {
                proxy_pass https://localhost:port;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "Upgrade";
                proxy_set_header Host $http_host;
                proxy_read_timeout 36000s;
        }
    }

**Important**

- For `proxy_pass`, the same address including port must be specified as was previously used by Docker.
- The official NGINX documentation states that the `if` rule for the redirect is not recommended. It is better to use the `return 301`. Therefore this has been commented out and added below.

## Setup in the Cloud
Now the data still has to be entered in the cloud. This is much easier than with OnlyOffice because only the URL has to be entered. After clicking on *Save*, the system directly checks whether the server is accessible.
