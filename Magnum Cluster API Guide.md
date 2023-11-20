Requirements:
- Openstack Cluster was deployed by Kolla-Ansbile# I tested on Openstack Yoga
- Octavia must work.
- Cinder(optional)

# Install K8S cluster management
We setup a k8s cluster to manage workload clusters which were created by Magnum.

We need get /root/.kube/config from cluster management.

# Magnum

We need log in(as root) to magnum api and magnum conductor and install 

```

apt update
pip install magnum-cluster-api
wget https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz # We can choose other vervion
tar -zxvf helm-v3.13.2-linux-amd64.tar.gz 
mv linux-amd64/helm /usr/local/bin/
```

We need helm to make this work. Because There is no helm in magnum containers.

Copy  /root/.kube/config from cluster management to /var/lib/magnum/.kube/config

* I tested on runtime container. If you want to build images. You must do above steps when use Kolla build. You need read followed link too.
  
  https://review.opendev.org/c/openstack/kolla-ansible/+/896292




