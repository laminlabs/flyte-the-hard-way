# Configure roles and service accounts

In order to restrict what Flyte components are entitled to do in an AWS environment, this guide leverages the integration between Service Accounts and IAM Roles. In this way, Flyte’s control plane components and data plane Pods (where the actual workflows run) will use Kubernetes service accounts associated with a set of permissions defined in the IAM Role. Any changes in the scope of permissions or policies in the IAM Role, will be inherited by the Kubernetes resources providing centralized control over Authorization:

![](./images/flyte-eks-permissions.png)

## Configure an OIDC provider for the EKS cluster

1. Verify that an OIDC issuer was created as part of the EKS deployment process:

```bash
aws eks describe-cluster --region <region> --name <Name-EKS-Cluster> --query "cluster.identity.oidc.issuer" --output text
```
2. Create the OIDC provider that will be associated with the EKS cluster:
```bash
eksctl utils associate-iam-oidc-provider --region <region> --cluster <Name-EKS-Cluster> --approve
```
3. From the AWS Management Console, verify that the OIDC provider has been created by going to **IAM** and then **Identity providers**. There should be a new provider entry has with the same <UUID-OIDC> issuer as the cluster’s.

## Create IAM Role
1. Create the `flyte-system-role` IAM role without creating a service account; it will be created by running the Helm chart at the end of the process:

```bash
eksctl create iamserviceaccount --cluster=fthw-eks-cluster --name=flyte-backend-flyte-binary --role-only --role-name=flyte-system-role --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --approve --region <region-code> --namespace flyte
```

2. Verify that the trust relationship between the IAM role and the OIDC provider has been created correctly:

```bash
aws iam get-role --role-name flyte-system-role --query Role.AssumeRolePolicyDocument
```
Example output:
```yaml
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::442400327471:oidc-provider/oidc.eks.<region-code>.amazonaws.com/id/<UUID-OIDC>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-1.amazonaws.com/id/<UUID-OIDC>:sub": "system:serviceaccount:flyte:flyte-backend-flyte-binary",
                    "oidc.eks.<region-code>.amazonaws.com/id/<UUID-OIDC>:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```
---
Next: [Create a relational database](04-create-database.md)
