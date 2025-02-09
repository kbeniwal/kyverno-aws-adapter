# Getting Started

This is a guide on how to get started with the Nirmata Kyverno Adapter for AWS. To learn more about Kyverno, check out the [official documentation](https://kyverno.io/docs/).

## Prerequisites

- a running EKS Cluster (refer to [Creating an Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html))
- AWS CLI (refer to [Installing the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
- eksctl (refer to [Installing eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html))
- Helm CLI (refer to [Installing Helm](https://helm.sh/docs/intro/install/))

## Installing the Nirmata Kyverno Adapter for AWS

There are a few steps we need to follow before installing the Helm chart.

### Creating the IAM Policy

To fetch the EKS Cluster configuration, the AWS Adapter needs the below permissions that has to be expressed via an IAM Policy.

```bash
cat >my-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement0",
            "Effect": "Allow",
            "Action": [
                "eks:AccessKubernetesApi",
                "eks:DescribeAddon",
                "eks:DescribeAddonVersions",
                "eks:DescribeCluster",
                "eks:DescribeFargateProfile",
                "eks:DescribeIdentityProviderConfig",
                "eks:DescribeNodegroup",
                "eks:DescribeUpdate",
                "eks:ListAddons",
                "eks:ListClusters",
                "eks:ListFargateProfiles",
                "eks:ListIdentityProviderConfigs",
                "eks:ListNodegroups",
                "eks:ListTagsForResource",
                "eks:ListUpdates"
            ],
            "Resource": [
                "arn:aws:eks:*:<account-id>:identityproviderconfig/*/*/*/*",
                "arn:aws:eks:*:<account-id>:fargateprofile/*/*/*",
                "arn:aws:eks:*:<account-id>:nodegroup/*/*/*",
                "arn:aws:eks:*:<account-id>:cluster/*",
                "arn:aws:eks:*:<account-id>:addon/*/*/*"
            ]
        },
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:DescribeFlowLogs",
                "ecr:DescribeRepositories",
                "inspector2:BatchGetAccountStatus"
            ],
            "Resource": "*"
        }
    ]
}
EOF
aws iam create-policy --policy-name <aws-policy-name> --policy-document file://my-policy.json
```

**Note:** Make sure your AWS CLI is configured correctly. Follow the [official guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html) for setting it up.

### Creating IAM OIDC provider

An IAM Open ID Connect provider for the cluster is required to provide the reference in the IAM Role.

```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve --region <region-code>
```

### Creating the IAM Role

Create an IAM Role that references the policy we created earlier.

**Note:** We will use `eksctl` in this guide to create the IAM Role. The serviceaccount creation will be done by the Helm chart so that appropriate RBAC is assigned by the chart itself. If you wish to use your own serviceaccount, then make sure all the necessary rolebinding and clusterrolebinding are present.

```bash
eksctl create iamserviceaccount --name nirmata-aws-adapter  --namespace nirmata-aws-adapter --cluster <cluster-name> --role-name nirmata-adapter-role --attach-policy-arn arn:aws:iam::<account-id>:policy/<aws-policy-name>   --role-only  --approve
```

This will create a new IAM Role that references the policy we created above. This also creates the trust-relationship for us. If you wish to create the IAM Role via the management console or the AWS CLI, make sure to create the trust-relationship so that the AWS Adapter can assume this Role to fetch EKS Cluster details.

### Installing the AWS Adapter Helm chart

First we need to set the values that are needed to install the Helm chart. You can either pass them as arguments via the [command line](https://helm.sh/docs/helm/helm_install/#helm-install) or set them in a [values file](https://helm.sh/docs/chart_template_guide/values_files/).

As an example, here are the minimum values that you need to set.

```bash
# cat myvalues.yaml
eksCluster:
  name: <cluster-name>
  region: <cluster-region>

roleArn: arn:aws:iam::<account-id>:role/nirmata-adapter-role

rbac:
  create: true
  serviceAccount:
    name: nirmata-aws-adapter
```

**Note:** Update the myvalues.yaml file as per information specific to your account and cluster.

We will install the adapter in nirmata-aws-adapter namespace. Create the namespace using ```kubectl create namespace nirmata-aws-adapter```.

**Note:** If the IAM Role was created using a different namespace name, then that name should be used instead of "nirmata-aws-adapter" while following the guide to ensure that the adapter is installed in the intended namespace and the IAM Role is associated with the correct namespace.

Now let's install the Helm chart.

```bash
helm repo add nirmata-kyverno-aws-adapter https://nirmata.github.io/kyverno-aws-adapter/
helm repo update nirmata-kyverno-aws-adapter

helm install kyverno-aws-adapter nirmata-kyverno-aws-adapter/kyverno-aws-adapter -f myvalues.yaml --namespace nirmata-aws-adapter
```

If everything goes well, you should see an output similar to this.

```bash
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "nirmata-kyverno-aws-adapter" chart repository
Update Complete. ⎈Happy Helming!⎈
NAME: kyverno-aws-adapter
LAST DEPLOYED: Fri Dec 16 22:01:35 2022
NAMESPACE: nirmata-aws-adapter
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Chart version: v0.1.0
Kyverno-aws-adapter version: v0.1.0

Thank you for installing kyverno-aws-adapter ! Your release is named kyverno-aws-adapter.

You can check the status of your configuration with:

    kubectl get awsacfg -n nirmata-aws-adapter kyverno-aws-adapter -o yaml
```

### Verifying the AWS Adapter installation

Perform the below steps to see if the adapter is installed correctly.

```bash
# check if the controller pod is running.
kubectl get pods -n nirmata-aws-adapter
NAME                                  READY   STATUS    RESTARTS   AGE
kyverno-aws-adapter-997f45bb9-c2z5j   1/1     Running   0          109m

# check the awsadapterconfig status.
kubectl get awsacfg -n nirmata-aws-adapter
NAME                 ...   CLUSTER NAME     ...  LAST POLLED STATUS
kyverno-aws-adapter  ...   cluster-name     ...  success
```

### Uninstalling the AWS Adapter Helm chart

To uninstall the AWS Adapter Helm chart, use the following command.

```bash
helm uninstall kyverno-aws-adapter --namespace nirmata-aws-adapter
```

This removes all the Kubernetes components associated with the chart and deletes the release.

The `awsadapterconfigs.security.nirmata.io` CRD created by this chart is not removed by default and should be manually cleaned up. So, after uninstalling helm chart the following command can be used to remove the CRD.

```bash
kubectl delete crd awsadapterconfigs.security.nirmata.io
```

### Deleting the AWSAdapterConfig

The `AWSAdapterConfig` CR is not deleted by `helm uninstall` or by deleting the pod, and must be manually cleaned up.

```bash
kubectl delete awsacfg kyverno-aws-adapter -n nirmata-aws-adapter
```
