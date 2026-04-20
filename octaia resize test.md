# Testing Load Balancer Resize with DevStack

This guide describes one way to test the load balancer resize feature on a
single-node DevStack deployment with the Amphora provider.

The resize API added by this change is:

```text
PUT /v2/lbaas/loadbalancers/{loadbalancer_id}/resize
```

It triggers a failover-based replacement of the amphorae and updates the load
balancer flavor when the operation completes successfully.

## Assumptions

- You are using a Linux VM for DevStack.
- You want to test the code from a branch or fork, not the upstream default.
- You are deploying Octavia with the Amphora provider.
- You are running a single-node DevStack environment.

## 1. Configure DevStack

Start from the sample file in `devstack/samples/singlenode/local.conf` and
point the Octavia plugin to the branch you want to test.

Example using a GitHub fork and the `review/890215` branch:

```ini
[[local|localrc]]
RECLONE=False

DATABASE_PASSWORD=password
ADMIN_PASSWORD=password
SERVICE_PASSWORD=password
RABBIT_PASSWORD=password

enable_plugin barbican https://opendev.org/openstack/barbican
enable_plugin neutron https://opendev.org/openstack/neutron
enable_plugin octavia https://github.com/nguyenhuukhoi/octavia.git review/890215

LIBS_FROM_GIT+=python-octaviaclient

enable_service rabbit
enable_service mysql
enable_service key

enable_service n-api
enable_service n-cpu
enable_service n-cond
enable_service n-sch
enable_service placement-api
enable_service placement-client

enable_service g-api

enable_service neutron
enable_service neutron-api
enable_service neutron-agent
enable_service neutron-dhcp
enable_service neutron-l3
enable_service neutron-metadata-agent
enable_service neutron-qos

enable_service octavia
enable_service o-api
enable_service o-cw
enable_service o-hk
enable_service o-hm
enable_service o-da
```

If you want to use a local checkout instead of a remote fork:

```ini
enable_plugin octavia file:///home/ubuntu/octavia
```

In that case, make sure the local repository is already checked out to the
branch that contains the resize patch before running `stack.sh`.

## 2. Run DevStack

```bash
git clone https://opendev.org/openstack/devstack /opt/stack/devstack
cp /path/to/local.conf /opt/stack/devstack/local.conf
cd /opt/stack/devstack
./stack.sh
```

After the deployment completes:

```bash
source /opt/stack/devstack/openrc admin admin
```

## 3. Create Nova flavors for the amphora instances

To verify that resize really changes the amphora shape, create two Nova flavors
with different sizes:

```bash
openstack flavor create amp.small --ram 1024 --disk 3 --vcpus 1 --private
openstack flavor create amp.large --ram 2048 --disk 5 --vcpus 2 --private
```

## 4. Create Octavia flavor profiles and flavors

Use the same load balancer topology for both flavors. For this test, only the
`compute_flavor` changes.

```bash
openstack loadbalancer flavorprofile create \
  --name resize-small-profile \
  --provider amphora \
  --flavor-data '{"compute_flavor":"amp.small","loadbalancer_topology":"SINGLE"}'

openstack loadbalancer flavorprofile create \
  --name resize-large-profile \
  --provider amphora \
  --flavor-data '{"compute_flavor":"amp.large","loadbalancer_topology":"SINGLE"}'

openstack loadbalancer flavor create \
  --name lb-small \
  --flavorprofile resize-small-profile \
  --description "LB on small amphora" \
  --enable

openstack loadbalancer flavor create \
  --name lb-large \
  --flavorprofile resize-large-profile \
  --description "LB on large amphora" \
  --enable
```

Save the Octavia flavor IDs:

```bash
SMALL_FLAVOR_ID=$(openstack loadbalancer flavor show lb-small -f value -c id)
LARGE_FLAVOR_ID=$(openstack loadbalancer flavor show lb-large -f value -c id)
```

## 5. Create a tenant network and load balancer

Create a test network:

```bash
openstack network create resize-net
openstack subnet create resize-subnet \
  --network resize-net \
  --subnet-range 192.168.50.0/24
```

Create a load balancer using the first Octavia flavor:

```bash
LB_ID=$(openstack loadbalancer create \
  --name resize-demo \
  --vip-subnet-id resize-subnet \
  --flavor "$SMALL_FLAVOR_ID" \
  -f value -c id)
```

Wait until it becomes `ACTIVE`:

```bash
watch -n 5 "openstack loadbalancer show $LB_ID -f value -c provisioning_status -c operating_status -c flavor_id"
```

## 6. Record the current amphora and Nova server

```bash
openstack loadbalancer amphora list $LB_ID -f yaml

AMPHORA_ID_BEFORE=$(openstack loadbalancer amphora list $LB_ID -f value -c id | head -n1)
COMPUTE_ID_BEFORE=$(openstack loadbalancer amphora show $AMPHORA_ID_BEFORE -f value -c compute_id)

openstack server show $COMPUTE_ID_BEFORE -f yaml | grep flavor
```

At this point, the Nova server should be using `amp.small`.

## 7. Call the resize API

The OpenStack client may not expose a dedicated resize command for this API, so
using `curl` is the simplest way to test it directly.

```bash
TOKEN=$(openstack token issue -f value -c id)
LB_ENDPOINT=$(openstack endpoint list --service load-balancer -f value -c URL | head -n1)

curl -i -X PUT "${LB_ENDPOINT}/v2/lbaas/loadbalancers/${LB_ID}/resize" \
  -H "X-Auth-Token: ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"new_flavor_id\":\"${LARGE_FLAVOR_ID}\"}"
```

Expected result:

- HTTP status `202 Accepted`

## 8. Verify the resize result

Wait for the load balancer to return to `ACTIVE`:

```bash
watch -n 5 "openstack loadbalancer show $LB_ID -f yaml | egrep 'provisioning_status|operating_status|flavor_id'"
```

Then inspect the new amphora and its Nova server:

```bash
openstack loadbalancer amphora list $LB_ID -f yaml

AMPHORA_ID_AFTER=$(openstack loadbalancer amphora list $LB_ID -f value -c id | head -n1)
COMPUTE_ID_AFTER=$(openstack loadbalancer amphora show $AMPHORA_ID_AFTER -f value -c compute_id)

openstack server show $COMPUTE_ID_AFTER -f yaml | grep flavor
```

Expected result:

- `flavor_id` on the load balancer is now `LARGE_FLAVOR_ID`
- the amphora `compute_id` changed
- the new Nova server flavor is `amp.large`

## 9. Negative test cases

### Invalid flavor ID

```bash
curl -i -X PUT "${LB_ENDPOINT}/v2/lbaas/loadbalancers/${LB_ID}/resize" \
  -H "X-Auth-Token: ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"new_flavor_id":"00000000-0000-0000-0000-000000000000"}'
```

Expected result:

- HTTP status `400`
- fault string similar to `Validation failure: Invalid flavor_id.`

### Topology mismatch

Create a flavor profile with `ACTIVE_STANDBY` and try to resize a `SINGLE` load
balancer to it:

```bash
openstack loadbalancer flavorprofile create \
  --name resize-asb-profile \
  --provider amphora \
  --flavor-data '{"compute_flavor":"amp.large","loadbalancer_topology":"ACTIVE_STANDBY"}'

openstack loadbalancer flavor create \
  --name lb-asb \
  --flavorprofile resize-asb-profile \
  --description "Topology mismatch test" \
  --enable

ASB_FLAVOR_ID=$(openstack loadbalancer flavor show lb-asb -f value -c id)

curl -i -X PUT "${LB_ENDPOINT}/v2/lbaas/loadbalancers/${LB_ID}/resize" \
  -H "X-Auth-Token: ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"new_flavor_id\":\"${ASB_FLAVOR_ID}\"}"
```

Expected result:

- HTTP status `400`
- a message indicating that the flavor is not compatible with the current load
  balancer topology

## 10. Useful logs

If the load balancer stays in `PENDING_UPDATE` or the resize takes too long,
watch these services:

```bash
sudo journalctl -u devstack@o-api -f
sudo journalctl -u devstack@o-cw -f
sudo journalctl -u devstack@o-hm -f
```

These are usually the most useful logs for:

- API validation failures
- provider driver calls
- failover execution
- amphora health state changes
