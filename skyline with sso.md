# Skyline

Need use domain name with ssl setup to make this lab works

## Setup Openstack cluster with Kolla-ansible

https://docs.openstack.org/kolla-ansible/yoga/ 

https://docs.openstack.org/kolla-ansible/yoga/admin/tls.html

## Setup Keystone

https://docs.openstack.org/skyline-console/latest/install/docker-install-ubuntu.html

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

docker exec -it 

