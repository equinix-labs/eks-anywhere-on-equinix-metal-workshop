<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 3: EKS-A installation and configuration

## Steps

### 1. Login to eksa-admin

Connect to the eksa-admin server via SSH and define required variables: `LC_POOL_ADMIN` (`${POOL_ADMIN}` value), `LC_POOL_VIP` (`${POOL_ADMIN}` value), `LC_TINK_VIP` (`${TINK_VIP}` value). 

> **_Pro Tip:_** The special args and environment setting in command below are just tricks to plumb $POOL_ADMIN and $POOL_VIP into the eksa-admin environment

```sh
LC_POOL_ADMIN=$POOL_ADMIN LC_POOL_VIP=$POOL_VIP LC_TINK_VIP=$TINK_VIP ssh -o SendEnv=LC_POOL_ADMIN,LC_POOL_VIP,LC_TINK_VIP root@$PUB_ADMIN
```

> **_Note:_** The remaining steps assume you have logged into `eksa-admin` with the SSH command shown above

### 2. Install EKS CLIs

[Install `eksctl` and the `eksctl-anywhere` plugin](https://anywhere.eks.amazonaws.com/docs/getting-started/install/#install-eks-anywhere-cli-tools) on eksa-admin.

```sh
curl "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
    --silent --location \
    | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
```

```sh
export EKSA_RELEASE="0.14.3" OS="$(uname -s | tr A-Z a-z)" RELEASE_NUMBER=30
curl "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/${RELEASE_NUMBER}/artifacts/eks-a/v${EKSA_RELEASE}/${OS}/amd64/eksctl-anywhere-v${EKSA_RELEASE}-${OS}-amd64.tar.gz" \
    --silent --location \
    | tar xz ./eksctl-anywhere
sudo mv ./eksctl-anywhere /usr/local/bin/
```

### 3. Install `kubectl`

```sh
snap install kubectl --channel=1.25 --classic
```

Version 1.25 matches the version used in the eks-anywhere repository.

<details><summary>Alternatively, install via APT.</summary>

```sh
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install kubectl
```
</details>

### 4. Install `Docker`

Run the docker install script:

```sh
curl -fsSL https://get.docker.com -o get-docker.sh 
chmod +x get-docker.sh
./get-docker.sh
```

Alternatively, follow the instructions from <https://docs.docker.com/engine/install/ubuntu/>.

### 5. Create EKS-A Cluster config

```sh
export TINKERBELL_HOST_IP=$LC_TINK_VIP
export CLUSTER_NAME="${USER}-${RANDOM}"
export TINKERBELL_PROVIDER=true
eksctl anywhere generate clusterconfig $CLUSTER_NAME --provider tinkerbell > $CLUSTER_NAME.yaml
```

> **_Note:_** The remaining steps assume you have defined the variables set above

### 6. Edit generated cluster config file

#### 6.1. Install yq

``` sh
snap install yq
```

#### 6.2. Generate a public SSH key and store it in a variable called 'SSH_PUBLIC_KEY'

```sh
ssh-keygen -t rsa -f /root/.ssh/id_rsa -q -N ""
export SSH_PUBLIC_KEY=$(cat /root/.ssh/id_rsa.pub)
```

#### 6.3. Make all necessary changes to the config cluster file

By running below `yq` command you will:

- Set control-plane IP `host` for Cluster resource
- Set the `tinkerbellIP` in the `TinkerbellDatacenterConfig` resource
- Set the public SSH key created in the previous step for each `TinkerbellMachineConfig`
- Set the `hardwareSelector` for each `TinkerbellMachineConfig` - `cp` or `worker`
- Change the `templateRef` for each `TinkerbellMachineConfig` section - We will add a `TinkerbellTemplateConfig` in next step

```sh
yq eval -i '
(select(.kind == "Cluster") | .spec.controlPlaneConfiguration.endpoint.host) = env(LC_POOL_VIP) |
(select(.kind == "TinkerbellDatacenterConfig") | .spec.tinkerbellIP) = env(LC_TINK_VIP) |
(select(.kind == "TinkerbellMachineConfig") | (.spec.users[] | select(.name == "ec2-user")).sshAuthorizedKeys) = [env(SSH_PUBLIC_KEY)] |
(select(.kind == "TinkerbellMachineConfig" and .metadata.name == env(CLUSTER_NAME) + "-cp" ) | .spec.hardwareSelector.type) = "cp" |
(select(.kind == "TinkerbellMachineConfig" and .metadata.name == env(CLUSTER_NAME)) | .spec.hardwareSelector.type) = "worker" |
(select(.kind == "TinkerbellMachineConfig") | .spec.templateRef.kind) = "TinkerbellTemplateConfig" |
(select(.kind == "TinkerbellMachineConfig") | .spec.templateRef.name) = env(CLUSTER_NAME)
' $CLUSTER_NAME.yaml
```

#### 6.4. Append the following `TinkerbellTemplateConfig` resource with the Tinkerbell settings to the config cluster file

```yaml
cat << EOF >> $CLUSTER_NAME.yaml
---
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: TinkerbellTemplateConfig
metadata:
  name: ${CLUSTER_NAME}
spec:
  template:
    global_timeout: 6000
    id: ""
    name: ${CLUSTER_NAME}
    tasks:
    - actions:
      - environment:
          COMPRESSED: "true"
          DEST_DISK: /dev/sda
          IMG_URL: https://anywhere-assets.eks.amazonaws.com/releases/bundles/29/artifacts/raw/1-25/bottlerocket-v1.25.6-eks-d-1-25-7-eks-a-29-amd64.img.gz
        image: public.ecr.aws/eks-anywhere/tinkerbell/hub/image2disk:6c0f0d437bde2c836d90b000312c8b25fa1b65e1-eks-a-29
        name: stream-image
        timeout: 600
      - environment:
          CONTENTS: |
            # Version is required, it will change as we support
            # additional settings
            version = 1

            # "eno1" is the interface name
            # Users may turn on dhcp4 and dhcp6 via boolean
            [enp1s0f0np0]
            dhcp4 = true
            dhcp6 = false
            # Define this interface as the "primary" interface
            # for the system.  This IP is what kubelet will use
            # as the node IP.  If none of the interfaces has
            # "primary" set, we choose the first interface in
            # the file
            primary = true
          DEST_DISK: /dev/sda12
          DEST_PATH: /net.toml
          DIRMODE: "0755"
          FS_TYPE: ext4
          GID: "0"
          MODE: "0644"
          UID: "0"
        image: public.ecr.aws/eks-anywhere/tinkerbell/hub/writefile:6c0f0d437bde2c836d90b000312c8b25fa1b65e1-eks-a-29
        name: write-netplan
        pid: host
        timeout: 90
      - environment:
          BOOTCONFIG_CONTENTS: |
            kernel {
                console = "ttyS1,115200n8"
            }
          DEST_DISK: /dev/sda12
          DEST_PATH: /bootconfig.data
          DIRMODE: "0700"
          FS_TYPE: ext4
          GID: "0"
          MODE: "0644"
          UID: "0"
        image: public.ecr.aws/eks-anywhere/tinkerbell/hub/writefile:6c0f0d437bde2c836d90b000312c8b25fa1b65e1-eks-a-29
        name: write-bootconfig
        pid: host
        timeout: 90
      - environment:
          DEST_DISK: /dev/sda12
          DEST_PATH: /user-data.toml
          DIRMODE: "0700"
          FS_TYPE: ext4
          GID: "0"
          HEGEL_URLS: http://${LC_POOL_ADMIN}:50061,http://${LC_TINK_VIP}:50061
          MODE: "0644"
          UID: "0"
        image: public.ecr.aws/eks-anywhere/tinkerbell/hub/writefile:6c0f0d437bde2c836d90b000312c8b25fa1b65e1-eks-a-29
        name: write-user-data
        pid: host
        timeout: 90
      - image: public.ecr.aws/eks-anywhere/tinkerbell/hub/reboot:6c0f0d437bde2c836d90b000312c8b25fa1b65e1-eks-a-29
        name: reboot-image
        pid: host
        timeout: 90
        volumes:
        - /worker:/worker
      name: ${CLUSTER_NAME}
      volumes:
        - /dev:/dev
        - /dev/console:/dev/console
        - /lib/firmware:/lib/firmware:ro
      worker: '{{.device_1}}'
    version: "0.1"
EOF
```

### 7. Create an EKS-A Cluster

Double check and be sure `$LC_POOL_ADMIN` and `$CLUSTER_NAME` are set correctly before running this (they were passed through SSH or otherwise defined in previous steps). Otherwise manually set them!

```sh
eksctl anywhere create cluster --filename $CLUSTER_NAME.yaml \
--hardware-csv hardware.csv --tinkerbell-bootstrap-ip $LC_POOL_ADMIN
```

### 8. Reboot Nodes

Steps to run locally while `eksctl anywhere` is creating the cluster

When the command above (step 7) indicates that it is `Creating new workload cluster`, reboot the two nodes. This
is to force them attempt to iPXE boot from the tinkerbell stack that `eksctl anywhere` command creates.

> **_Note:_** that this must be done without interrupting the `eksctl anywhere create cluster` command

Option 1 - You can use this command to automate it, but you'll need to be back on the original host.

```sh
node_ids=$(metal devices list -o json | jq -r '.[] | select(.hostname | startswith("eksa-node")) | .id')
for id in $(echo $node_ids); do
  metal device reboot -i $id
done
```

Option 2 - Instead of rebooting the nodes from the host you can force the iPXE boot from your local by
[accessing each node's SOS console](https://metal.equinix.com/developers/docs/resilience-recovery/serial-over-ssh/).
You can retrieve the uuid and facility code of each node using the metal cli, UI Console or the Equinix Metal's API.
By default, any existing ssh key in the project can be used to login.

```sh
ssh {node-uuid}@sos.{facility-code}.platformequinix.com -i </path/to/ssh-key>
```

> **_Note:_** After rebooting nodes the `eksctl anywhere create cluster` command output will hang at `Creating new workload cluster` for almost 20 min without any further feedback

### 9. Confirm Success

After 20-30 min you will see the below logs message if the whole process is successful

```sh
Installing networking on workload cluster
Creating EKS-A namespace
Installing cluster-api providers on workload cluster
Installing EKS-A secrets on workload cluster
Installing resources on management cluster
Moving cluster management from bootstrap to workload cluster
Installing EKS-A custom components (CRD and controller) on workload cluster
Installing EKS-D components on workload cluster
Creating EKS-A CRDs instances on workload cluster
Installing GitOps Toolkit on workload cluster
GitOps field not specified, bootstrap flux skipped
Writing cluster config file
Deleting bootstrap cluster
:tada: Cluster created!
--------------------------------------------------------------------------------------
The Amazon EKS Anywhere Curated Packages are only available to customers with the
Amazon EKS Anywhere Enterprise Subscription
--------------------------------------------------------------------------------------
Enabling curated packages on the cluster
Installing helm chart on cluster	{"chart": "eks-anywhere-packages", "version": "0.2.30-eks-a-29"}
```

### 10. Verify the nodes are deployed properly

To verify the nodes are deployed properly, set the generated cluster kubeconfig file as the default k8s config

```sh
cp /root/$CLUSTER_NAME/$CLUSTER_NAME-eks-a-cluster.kubeconfig /root/.kube/config
```

You can run now below commands to check nodes and pods in cluster

```sh
kubectl get nodes -o wide
kubectl get pods -A
```

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* Can I scale up Nodes after initial setup?
