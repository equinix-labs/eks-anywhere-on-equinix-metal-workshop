<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 4: Expanding a cluster

In this section we will add new node of the exact same type as the previous nodes to the cluster. For example, if you use project defaults you'll want to add a m3.small.x86 as the new node. Also, this example is just adding a new worker node for simplicity. Adding control plane nodes is possible, but requires thinking through how many nodes are added as well as labeling them as `type=cp` instead of `type=worker`.

## Steps

### 1. Deploy an additional node

Locally create a new device using metal-cli

```sh
NEW_HOSTNAME="your new hostname"
POOL_ADMIN="IP address of your admin machine"
metal device create --plan m3.small.x86 --metro da --hostname $NEW_HOSTNAME 
--ipxe-script-url http://$POOL_ADMIN/ipxe/ --operating-system custom_ipxe
```

Note the device's UUID from the command output to use it in next steps

### 2. Network configuration

> **_Pro Tip:_** If you forgot to note the device ID you can use `metal device get` to list and retrieve your devices information

```sh
DEVICE_ID="<<UUID you noted above>>"
BOND0_PORT=$(metal devices get -i $DEVICE_ID -o json  | 
jq -r '.network_ports [] | select(.name == "bond0") | .id')
ETH0_PORT=$(metal devices get -i $DEVICE_ID -o json  | 
jq -r '.network_ports [] | select(.name == "eth0") | .id')
```

```sh
VLAN_ID="Your VLAN ID, likely 1000"
metal port convert -i $BOND0_PORT  --layer2 --bonded=false --force
metal port vlan -i $ETH0_PORT -a $VLAN_ID
```

### 3. Build a new inventory file

Next, back from the eks-admin device, put the following in a new csv file `hardware2.csv`

```csv
hostname,mac,ip_address,gateway,netmask,nameservers,disk,labels
<HOSTNAME>,<MAC_ADDRESS>,<IP>,<GATEWAY>,<NETMASK>,8.8.8.8|8.8.4.4,/dev/sda,type=worker
```

### 4. Add the node to EKS-A

Get your machine deployment group name:

```sh
kubectl get machinedeployments -n eksa-system
```

Generate the kubernetes yaml from your hardware2.csv file:

```sh
eksctl anywhere generate hardware -z hardware2.csv > cluster-scale.yaml
```

Edit cluster-scale.yaml and remove the two bmc items.

Use the machinedeployment group name along with the csv file to scale the cluster.

```sh
kubectl apply -f cluster-scale.yaml
kubectl scale machinedeployments -n eksa-system <Your MachineDeployment Group Name> --replicas 1
```

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* Can I scale down Nodes?
