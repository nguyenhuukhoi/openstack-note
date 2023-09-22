# Skyline
Openstack Yoga.

Need use domain name with ssl setup to make this lab works

## Setup Google Callback

Go to Goolge console to setup API key and URI call back

Authorized JavaScript origins: https://youropenstackdomain

Authorized redirect URIs: https://youropenstackdomain/redirect_uri

## Setup Openstack cluster with Kolla-ansible

https://docs.openstack.org/kolla-ansible/yoga/ 

https://docs.openstack.org/kolla-ansible/yoga/admin/tls.html

## Setup Keystone

You should these files:

/etc/kolla/config/keystone/mapping/accounts.google.com.provider

use content of file below

https://accounts.google.com/.well-known/openid-configuration

/etc/kolla/config/keystone/mapping/accounts.google.com.client
````
{
    "client_id": "google client id",
    "client_secret": "google client secret"
}
````
/etc/kolla/config/keystone/mapping/accounts.google.com.conf
````
{}
````
nano /etc/kolla/globals.yml

````
keystone_identity_providers:
  - name: "myidp"
    openstack_domain: "default"
    protocol: "openid"
    identifier: "https://accounts.google.com"
    public_name: "Authenticate via Google SSO"
    attribute_mapping: "mappingId1"
    metadata_folder: "/etc/kolla/config/keystone/mapping"
keystone_identity_mappings:
  - name: "mappingId1"
    file: "/etc/kolla/config/keystone/mapping/google.json"
````
nano /etc/kolla/config/keystone.conf

````
[federation]
trusted_dashboard = https://domainame.fpt.net:9999/api/openstack/skyline/api/v1/websso
````
*** domainname which mapped to skyline host

## Setup Skyline

https://docs.openstack.org/skyline-console/latest/install/docker-install-ubuntu.html

### SSL for Skyline

docker exec -it [your skyline container] /bin/bash
````
apt install nano
````
nano /etc/nginx/nginx.conf

````
        listen 0.0.0.0:9999 default_server ssl http2;

        root /usr/local/lib/python3.8/dist-packages/skyline_console/static;

        listen 0.0.0.0:9999 default_server ssl http2;
        # Add index.php to the list if you are using PHP
        index index.html;

        server_name skylinedomainname;
        #Your cert path
        ssl_certificate "/etc/ssl/ha.pem";
        ssl_certificate_key "/etc/ssl/hain.pem";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        error_page 497 https://$http_host$request_uri;

        location / {
            # First attempt to serve request as file, then
            # as directory, then fall back to displaying a 404.
            try_files $uri $uri/ /index.html;
            expires 1d;
            add_header Cache-Control "public";
        }

