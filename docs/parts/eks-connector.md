<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 4: Connect the cluster to EKS with EKS Connector

This section covers the basic steps to connect your cluster to EKS with the EKS Connector. There are many more details (include pre-requisites like IAM permissions) in the [EKS Connector Documentation](https://docs.aws.amazon.com/eks/latest/userguide/eks-connector.html).

## Steps

### 1. Setup AWS credentials

Follow the [AWS documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html) and set the environment variables with your authentication info for AWS in your eks-admin device. For example:

```sh
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-west-2
```

### 2. Register the cluster

Use `eksctl` to register the cluster

```sh
CLUTER_NAME="My-Metal-Cluster"
AWS_REGION="eu-west-1"
eksctl register cluster --name $CLUTER_NAME --provider other --region region-code
```

If it succeeded, the output will show several .yaml files that were created and need to be registered with the cluster. For example, at the time of writing, applying those files would be done like so:

```sh
kubectl apply -f eks-connector.yaml,eks-connector-clusterrole.yaml,eks-connector-console-dashboard-full-access-group.yaml
```

Even more info can be found at the [eksctl documentation](https://eksctl.io/usage/eks-connector/).

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* Is there any official alternative?
