============
 OpenSlides Installation
============

Requirements
============

- A Jitsi Server exposing the following endpoints (NEEDS CHECK!)
    - /external_api.js
    - /external_api.min.js
    - /external_api.map
    - /external_api.min.map


For a production deployment with around ~1k concurrent users you should at least have the following:
- 8 Cores
- 16 GB RAM
- 50 GB SSD
- Public IPv4 address
- Public IPv6 address


Installation
============

The main deployment method is using Git, Docker and Docker Compose. You only need
to have these tools installed and no further dependencies. If you want a simpler
setup or are interested in developing, please refer to `development
instructions <DEVELOPMENT.rst>`_.

Get OpenSlides
--------------

First, you have to clone this repository::

    git clone https://github.com/OpenSlides/OpenSlides.git --recurse-submodules
    cd OpenSlides/

**Note about migrating from version 3.3 or earlier**: With OpenSlides 3.4 submodules
and a Docker setup were introduced. If you ran into problems try to delete your
``settings.py``. If you have an old checkout you need to check out the current master
first and initialize all submodules::

    git submodule update --init

Setup Docker images
-------------------

You need to build the Docker images and have to setup some configuration. First,
configure HTTPS by checking the `Using HTTPS`_ section. In this section are
reasons why HTTPS is required for large deployments.

Go to ``docker`` subdirectory::

    cd docker

Then build all images with this script::

    ./build.sh all

You must define a Django secret key in ``secrets/django.env``, for example::

    printf "DJANGO_SECRET_KEY='%s'\n" \
      "$(tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 64)" > secrets/django.env




We also strongly recommend that you set a secure admin password but it is not
strictly required. If you do not set an admin password, the default login
credentials will be displayed on the login page. Setting the admin password::

    cp secrets/adminsecret.env.example secrets/adminsecret.env
    vi secrets/adminsecret.env

If you're lazy you can reuse the same routine as above like this:

    printf "OPENSLIDES_ADMIN_PASSWORD='%s'\n" \
     "$(tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 64)" > secrets/adminsecret.env

Afterwards, generate the configuration file::

    m4 docker-compose.yml.m4 > docker-compose.yml

Finally, you can start the instance using ``docker-compose``::

    docker-compose up

OpenSlides is accessible on http://localhost:8000/ (or https, if configured).

Use can also use daemonized instance::

    docker-compose up -d
    docker-compose logs
    docker-compose down

Using in production
-----------

If you want to use OpenSlides in production, you should'nt expose the frontend via HTTP.
You can get a free certificate via Let's Encrypt.

Based on the assumption you're using a Debian based OS (debian/Ubuntu/etc.) you need to install the following packages:

- certbot
(If you want to have a redirect from port 80 (HTTP) to port 443 (HTTPS) consider installing nginx too. It also maked the revalidation of the TLS certs much easier)
- nginx
- python3-certbot-nginx

Generate a certificate with ``certbot --nginx -d <DOMAIN NAME>``.

After you've genereated the certificate you can symlink it into ``~/OpenSlides/caddy/certs`` like so:
- ``ln -s /etc/letsencrypt/live/<DOMAIN NAME>/fullchain.pem ~/OpenSlides/caddy/certs/cert.pem``
- ``ln -s /etc/letsencrypt/live/<DOMAIN NAME>/privkey.pem ~/OpenSlides/caddy/certs/key.pem``

After that, you need to rebuild the webserver with ``./build.sh proxy```

Edit the ``.env`` file in the docker directory to you liking.
You need to edit the following lines:
- INSTANCE_DOMAIN=<DOMAIN NAME>
- EXTERNAL_HTTP_PORT=443

Run ``( set -a; source .env; m4 docker-compose.yml.m4 ) > docker-compose.yml``.

Edit the following lines in the docker-compose.yml file:

- JITSI_DOMAIN
- JITSI_ROOM_PASSWORD
- JITSI_ROOM_NAME

In the "services" area edit the following:

``ports:
      - "127.0.0.1:443:8000"``

to:
``ports:
      - "443:8000"``

If you want to use the voting feature set ``ENABLE_ELECTRONIC_VOTING`` to true.

If you want to use the chat feature set ``ENABLE_CHAT``to true.


Run ``docker-compose up -d`` and you should be ready to go.

(If you used nginx in certificate generation it is likely to fail due to port 443 being already in use. In that case you want to do the following:

``rm /etc/nginx/sites-enabled/default``

``vim /etc/nginx/sites-enabled/<DOMAIN-NAME>```

Insert the following:

``server {
	listen 80;
	listen [::]:80;
	server_name <DOMAIN NAME>;
        return 301 https://$host$request_uri;
}``

Run ``nginx -t`` to test the config an ``nginx -s reload`` to apply the config.

Now run ``docker-compose up -d`` again.

You should be able to login to OpenSlides under "https://<DOMAIN NAME>".
(The credentials for the admin account are stored in ``~/OpenSlides/docker/secrets/adminsecret.env``)

If there are problems with your instance check ``docker-compose logs -f`` for a production log of your OpenSlides installation.
