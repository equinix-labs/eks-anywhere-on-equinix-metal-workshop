<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 4: Expanding a cluster

In this section we will add new node of the exact same type as the previous nodes to the cluster. For example, if you use project defaults you'll want to add a m3.small.x86 as the new node. Also, this example is just adding a new worker node for simplicity. Adding control plane nodes is possible, but requires thinking through how many nodes are added as well as labeling them as `type=cp` instead of `type=worker`.

## Steps

### 1. Deploy an additional node

Locally create a new device using metal-cli

```sh
NEW_HOSTNAME="new-eksa-node"
POOL_GW=$(metal ip list -o json | jq -r '.[] | select(.tags | contains(["eksa"]))? | .gateway')
POOL_ADMIN=$(python3 -c 'import ipaddress; print(str(ipaddress.IPv4Address("'${POOL_GW}'")+1))')
metal device create --plan m3.small.x86 --metro da --hostname $NEW_HOSTNAME --ipxe-script-url http://$POOL_ADMIN/ipxe/ --operating-system custom_ipxe
```

Note the device's UUID from the command output to use it in next steps

```sh
NEW_DEVICE_ID=$(metal devices list -o json | jq -r '.[] | select(.hostname | startswith('\"$NEW_HOSTNAME\"')) | .id')
```

### 2. Network configuration

> **_Pro Tip:_** If you forgot to note the device ID you can use `metal device get` to list and retrieve your devices information

```sh
BOND0_PORT=$(metal devices get -i $NEW_DEVICE_ID -o json |
jq -r '.network_ports [] | select(.name == "bond0") | .id')
ETH0_PORT=$(metal devices get -i $NEW_DEVICE_ID -o json |
jq -r '.network_ports [] | select(.name == "eth0") | .id')
```

```sh
VLAN_ID="1000"
metal port convert -i $BOND0_PORT --layer2 --bonded=false --force
metal port vlan -i $ETH0_PORT -a $VLAN_ID
```

### 3. Build a new inventory file

Next, back from the eks-admin device, generate a new hardware csv file `hardware2.csv`

```sh
echo "hostname,vendor,mac,ip_address,gateway,netmask,nameservers,disk,labels" > hardware2.csv
NEW_DEVICE_MAC=$(metal device get -i $NEW_DEVICE_ID -o json | jq -r '.network_ports | .[] | select(.name == "eth0") | .data.mac')
POOL_NM=$(metal ip list -o json | jq -r '.[] | select(.tags | contains(["eksa"]))? | .netmask')
NEW_DEVICE_IP=$(python3 -c 'import ipaddress; print(str(ipaddress.IPv4Address("'${POOL_GW}'")+'4'))')
echo "$NEW_HOSTNAME,Equinix,${NEW_DEVICE_MAC},${NEW_DEVICE_IP},${POOL_GW},${POOL_NM},8.8.8.8|8.8.4.4,/dev/sda,type=worker" >> hardware2.csv
```

Copy `hardware2.csv` to `eksa-admin`:

```sh
PUB_ADMIN=$(metal devices list  -o json  | jq -r '.[] | select(.hostname=="eksa-admin") | .ip_addresses [] | select(contains({"public":true,"address_family":4})) | .address')
scp hardware2.csv root@$PUB_ADMIN:/root
```

### 4. Add the node to EKS-A

Login to eksa-admin

```sh
ssh root@$PUB_ADMIN:/root
```

Generate the kubernetes yaml from your `hardware2.csv` file

```sh
eksctl anywhere generate hardware -z hardware2.csv > cluster-scale.yaml
```

NOTE Edit cluster-scale.yaml and remove the two bmc resources

Get your machine deployment group name

```sh
MD_GROUP=$(kubectl get machinedeployments -n eksa-system -o jsonpath='{.items[0].metadata.name}')
```

Use the machinedeployment group name along with the csv file to scale the cluster.

```sh
kubectl apply -f cluster-scale.yaml
kubectl scale machinedeployments -n eksa-system --replicas $MD_GROUP
```

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* Can I scale down Nodes?
