# aws-iam-authenticator-k3d-demo
Quick and sloppy demo setting up aws-iam-authenticator on k3d.

## Steps

- Install aws-iam-authenticator
- Create an IAM role that can be assumed with your AWS creds
- Set this role arn as the value in them helm chart
- Set your volume mount paths in the k3d-config.yaml to your local machine (TODO: set this as an env var)


Create a local cluster with k3d and deploy the helm chart

```bash
k3d cluster create --config k3d-config.yaml

# Load docker image into k3d
docker pull 602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon/aws-iam-authenticator:v0.5.9
docker tag 602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon/aws-iam-authenticator:v0.5.9 aws-iam-authenticator:v0.5.9
k3d image import -c test-aws aws-iam-authenticator:v0.5.9

# Deploy aws-iam-authenticator
helm install aws-iam-authenticator
```

## Test

Set your kubeconfig user and set the context to use this user:

```yaml
users:
- name: kubernetes-admin
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "REPLACE_ME_WITH_YOUR_CLUSTER_ID"
        - "-r"
        - "REPLACE_ME_WITH_YOUR_ROLE_ARN"
```
